---
title: Fedy’s installation of Brackets bricks your Fedora installation
author: Andrea Veri
type: post
date: 2014-04-02T14:11:27+00:00
url: /2014/04/02/fedy-installation-of-brackets-bricks-your-fedora-installation/
categories:
  - Planet Fedora
  - Planet GNOME
tags:
  - fedora 20
  - fedy

---
I wanted to give <a href="http://satya164.github.io/fedy/" target="_blank">Fedy</a> a try yesterday, specifically to install the **Brackets **code editor designed for web developers. I&#8217;m pretty lazy when it comes to install external packages (from the Brackets.io&#8217;s homepage it looked like only a DEB file was available) and after asking a few friends who made heavy use of Fedy in the past about its stability and credibility I went ahead and followed the provided instructions to set it up.

The interface was pretty straightforward and installing Brackets was as easy and clicking on the relevant button. Before starting the installation I gave a fast look around to the various bash scripts used by Fedy to install the package I wanted and yeah, I admit I did not pay enough attention to a few lines of the code and went ahead with the installation.

After hacking a bit with Brackets I decided it was time for me to head to bed but shutting down my laptop surprisingly returned various errors related to Systemd&#8217;s journal not being able to shutdown properly. I then tried to reboot the machine and found out the laptop was totally not bootable anymore.

The error it was reported at boot (Systemd Journal not being able to start properly) was pretty strange and after looking around the web I couldn&#8217;t find any other report about similar failures. I then started digging around with a friend and made the following guesses:

  1. The root partition was running out of space (just 70M left), I then cleaned it a bit and rebooted with no luck. My first guess was _/tmp_ going out of space when Systemd tries to populate it at boot time.
  2. I checked yum history to find out what Fedy could have taken in, but nothing relevant was found given Fedy does not install RPM packages on its own, it usually retrieves a tarball (or in my case a DEB package) and installs it by extracting / cping the content
  3. I turned SELinux to Permissive and rebooted the machine, surprise, the machine was bootable again

The next move was running a _restorecon -r -v /_ against the root partition, the result was awful: the whole _/usr_&#8216;s context was turned into _usr\_tmp\_t_. Digging around the code for the Brackets installer the following code was found:

{{< highlight Bash >}}mkdir -p "${file%.*}"
ar p "$file" "data.tar.gz" | tar -C "${file%.*}" -xzf -
cp -af ${file%.*}/* "/"{{< / highlight >}}

<br\>

And previously:

{{< highlight Bash >}}get_file_quiet "http://download.brackets.io/" "brackets.htm"
get=$(cat "brackets.htm" | tr ' ' '\n' | grep -o "file.cfm?platform=LINUX${arch}&build=[0-9]*" | head -n 1 | sed -e 's/^/http:\/\/download.brackets.io\//')
file="brackets-LINUX.deb"{{< / highlight >}}

<br\>

So what the installer was doing is:

  1. Downloading the DEB file from the Brackets website
  2. Extracting its content to _/tmp/fedy_ and copying the contents of the _data.tar.gz_ tarball in place

A tree view of the _data.tar.gz_ file:

{{< highlight Bash >}}.
|-- opt
| `-- brackets
`-- usr
|-- bin
`-- share{{< / highlight >}}

<br\>

Copying the extracted content of the _data.tar.gz_ tarball to the target directories will do exactly one thing: it will overwrite the SELinux context of your _/usr_, _/bin_, _/share_ directories breaking your system. I would advise everyone to **NOT** make use of **Fedy** for installing the Brackets editor until the issue has been fixed. Honestly speaking I didn&#8217;t have time / willingness to check other bash scripts but something nasty might be found there as well. Generally I would never recommend to install anything on your system without making use of an RPM package. Lesson learned for me to never trust such tools in the future on my local system.

The issue seems it was reported already one month ago, we added our report to the same issue. You can track it at <a href="https://github.com/satya164/fedy/issues/79" target="_blank">https://github.com/satya164/fedy/issues/79</a>.

Resources:

  1. Faulty bash script: <a href="https://github.com/satya164/fedy/blob/master/plugins/soft/adobe_brackets.sh" target="_blank">https://github.com/satya164/fedy/blob/master/plugins/soft/adobe_brackets.sh</a>
  2. Why _usr\_tmp\_t_ gets added as _/usr_&#8216;s context: <a href="https://github.com/satya164/fedy/blob/master/fedy#L26" target="_blank">https://github.com/satya164/fedy/blob/master/fedy#L26</a>.
