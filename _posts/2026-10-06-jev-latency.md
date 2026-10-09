---
layout: post
title: "Latency and Accuracy in System One models"
date: 2026-10-06 13:56:00 -0700
tags:  code, models, short-life-span 
---

I've been a fan of Jev for a couple of weeks as a quick classifier for pre-routing agent requests. 
Today OpenAI released the beta of its [Decisions API](https://developers.openai.com/api/docs/guides/decisions), so I ran both through the same classifier test for accuracy and latency.

On a dataset of ~300 prompts:
- Accuracy: OpenAI 97%, Jev 96.5%. Effectively a tie.
- Latency p50: Jev ~95 ms, OpenAI ~153 ms
- Latency p95: Jev ~165 ms, OpenAI ~380 ms

Both are plenty fast for routing. I expect OpenAI will  get faster; their API today is just serving the Luna model, unoptimized.
Use either with confidence, and compare the features in-depth to match your needs (e.g. OpenAI supports image input, I am not using that so I did not factor it). 

I am sure there is lots of possible variance: I tested from Seattle on a low OpenAI usage tier. Want to run it yourself? The source for test is [here](https://github.com/jaimerodriguez/jev_openai_decisions_sample).  I am not a perf engineer, so suggestions for improvements (or mistakes) are welcome. 
 
Note: the dataset in the public sample is much smaller than the one I tested with, but it is representative. Plug in your own.