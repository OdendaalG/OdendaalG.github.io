+++
date = '2025-07-06T09:43:33+01:00'
draft = true
title = 'Where Did the Memory Go'
+++

# Background
In a recent project at a client there was a Python data manipulation app service built in Azure. The ask is simple: take data from an Azure storage account, wrangle it a bit, and store it in a database. Simple ETL. We had built it to run in a step-wise manner, one step taking the input of the previous and giving its output to the next.

The issue was memory inefficiency, our Azure function app kept crashing because it ran out of memory. To monitor this and understand where the memory went (which operation) we started blotting Python's own memory analysis package around:

```Python
{{ Python code on measuring peak and average memory }}#TODO
```

Armed with the above piece of code, and the step-wise nature of the program we constructed something similar to this:

```Python
{{ Python code on how memory measurement worked }}#TODO 
```

Done and done, now we'll be able to see which operation is eating the memory and crashing out application. We did. But it alluded to another issue entirely. Our maximum memory usage sat at around {{ Maximum memory usage }} while the Azure memory graph was measuring {{ Azure memory usage }}. Nearly double. #TODO

Where did the memory go and why could Python not see it?

# Possible causes
- Memory measurement tool used in Python is inadequate.
- Azure lies about the amount of memory it gives the containers.
- The container itself uses a large amount of memory additional to the actual program.

# Experimental setup
Python program that loads a file of predetermined size into memory and writes it to output.
Three measurement tools within Python.
Measurement tools within the OS docker container.
Local and deployed testing for gathering logs.
Build, deploy, log, destroy autonomously using golang code.

