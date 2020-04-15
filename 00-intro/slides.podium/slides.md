class: title
# <br>.thick[Deploying Django on Serverless Infrastructure]
## .thick[PyCon 2020 Tutorial]
![Image](images/footer.svg)
---

class: title
# Welcome!

???

Hello! It's great to see you.

Sorry, just a little bit of remote presentation humour there. I can't actually see you.

This is Deploying Django on Serverless Infrastrucutre.

---

class: title
# Agenda
### glasnt.com/unicodex-tutorial

???

We're going to be spending a bit of time together today, so here's a description.

We'll start with a short introduction, then we'll work through the practicals of our tutorial:

the setup
the x
the y

And then I'll have some parting words.

We'll be using the resources up at glasnt.com/unicodex-tutorial, which will include notes about the timestamps for each section if you want to skip forward or back at any time.

By the end of this tutorial, you will have deployed an example Django application on serverless infrastructure. Nice!

---

class: title
# Introduction

## What is deployment, anyway?
???

So, when I say deployment, what do I mean by that? What is deployment anyway?

---
class: title
## &nbsp;
### Here's a talk I prepared earlier:

## "What is deployment, anyway?"
### PyCon 2020, PyGotham 2019

???

I have a talk at PyCon 2020 about just that. I'll summarize it in a moment, but if you want to watch the full recording, you can look up the recording from either PyCon 2020, or from PyGotham 2019. It's a half hour talk where I go through what is required to deploy a django project installed for the startproject template, and the mandatory requirements it has.

---

class: title
# Django is stateful
## It requires<br>a database and static assets<br>out of the box.

???

in order for a django project to work, just a base project with a working django admin, you must have a database and you must have static assets.

Therefore there is a mimimum mandatory set of requirements.

---

class: title
# Stateful applications are complex

???

compare a stateful Django application to a stateless Flask application. You don't *need* a database for a basic Hello World flask application to work.

A stateless application is easier to deploy than a stateful one.

Stateful applications aren't *complicated*, per say, just complex.

But the complexity is lovely.



---

class: title
# .quote["Every production setup<br>will be a bit different"]
### - .prokyon[django] documentation


???

The django documentation itself, hiding in the page about deploying static to production, has this line.

Every production setup will be a bit different.

No setup is going to be the same. The variation of setups and the varitey of tools you can use exist for many reasons.

If you are in a company that has a mysql dba, you'll probably use mysql. If you have a coworker with anisble experience, you'll probably use that.

There is no one way to deploy your django application.

So any one method will be *a* way to deploy your application, but it may not be the way you deploy *your* applciation.

---

class: title
## What<br>do you want<br>to worry about?
???

what do you want to worry about? The tools you choose have to be right for you, because you are the one that has to maintain your project. Leaning on existing skills is a great way to reduce overhead because you don't have to learn an entirely new technology.

And with the worry-factor comes the discussion about using different types of platforms.

---

class: title
# Managed hosting

## An extraordinarily good idea

???

while you could deploy your project on a custom bit of physical server hardware in your own datacenter where you provision each component seperately and bespokely, we are at a stage in tech now that cloud based infrastrusture, or managed hosting, allows you to pay a little bit of money to have these things managed for you.

Cloud storage allows you to store your images and such in the cloud. Managed databases automate updates, maintenance and backups for you. Managed compute allows you to not have to worry about server updates and resets.

Many of these services have smaller free tiers to help you get started, but they are now robust commodity items that means that you don't have to spend your time being a system adminitrator or a dba, you can spend your time being a developer.

---

class: title
# Today's deployment
## Django on Google Cloud

???

given all that, today we'll be deploying an example django application on Google Cloud.

Google Cloud is one of the many cloud providers out there -- Amazon and Azure being others -- but we'll be using it today to provision and deploy our Django application.

---

class: title
## Example Application
### .smol[github.com/GoogleCloudPlatform/django-demo-app-unicodex]

???

We'll be using a sample application called "unicodex", which can be found on github.com under the GoogleCloudPlatform organisation.


This repo also contains documentation to deploy this sample application, which we will be using throughout this tutorial.


before we start spinning up cloud components, we'll look at our example application, test it out locally, and see just what it does.


---

class: title
## Onto part one!