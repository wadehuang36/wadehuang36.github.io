---
layout: post
title: "Email Templating Via SendGrid"
description: A good way to send emails.
tags: 
    - smtp
---

Sending Email is a very easy activity, you just use an SMTP Client and send parameters to an SMTP Server. Then, the email will be delivered to recipients. But once there are many kind of email have to send. Managing templates is problematic, your have to build a backend, then administrators can log in the backend to change templates. It might spend 1 or 2 days to build each backends. Therefore, this article will tell you how to use the template management of SendGrid to save your time.

## Create a template
[SendGrid](https://sendgrid.com/) provides many email related services which includes template management. 

In the dashboard, click Template > Transaction.
<img src="/assets/images/07da6e0437f3d7afd45c68b99825bc31ef637642.png" width="186" height="314">

Then, click `create a template` button and edit the content.
<img src="/assets/images/e34db772269bab0d0dfa71470d0805ad9f1e6957.png" width="690" height="323">


## Variables
you can define any variables in the template. SendGrid doesn't restrict the notation of variable, you can use any format, it matched the name in the parameters of send. In the above example, I use $ for the notation. (Actually, it uses like string.replace(name, value) method to replace text, so the notation is not necessary, but it helps reading).

## Send Email
SendGrid provides WebAPI and SMTP two methods to send emails. In the below example, I use WebAPI because it is easier to use.
<img src="/assets/images/802a5a6dd71c6c2d10c1359bbd0d1bf078f8dc78.png" width="577" height="485">
the values in the `substitutions` will replace the text in the template. For example, the value of `$system` will replace the `$system` in the template. 

## Benefits
Now the administrators can go to SendGrid to changes templates, so they won't call you if they want to change a word. Also, you don't need to build your own template management.

## References
- https://sendgrid.com/docs/API_Reference/Web_API_v3/Mail/index.html