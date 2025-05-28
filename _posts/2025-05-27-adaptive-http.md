# Adaptable Infrastructure on AWS: Combining ECS and Lambda Behind an ALB

I recently attended a session given by [Allen Helton (AWS Hero)](https://www.readysetcloud.io/blog/) on the past and future of IaC (infrastructure as code). In it he talked about a future where the infrastructure was more adaptive, allowing you to write code once and have it modify how it runs automatically. This brought me back to something I did about 7 years ago, where I had the same code running in both a container and in Lambda. It wasn't quite what Allen was talking about because it was a deploy time decision as to how it would run. That got me thinking about what it would take to make something like that work automatically. In this post, I’ll walk through an architecture that leverages both Amazon ECS and AWS Lambda behind a single Application Load Balancer (ALB), enabling you to dynamically shift traffic and infrastructure depending on usage patterns, all while running the same Node.js/Express codebase. I don't believe this is the end state that Allen was speaking of - I believe it needs to be easier and more automated (less code the developers have to write) - but I think this is an interesting start down that path.

---

## 💡 The Challenge

You have a Node.js application, and you want to serve it efficiently:

- During high traffic, use ECS to handle concurrency and throughput cost-effectively.
- During low traffic, save costs by scaling down ECS and using Lambda instead.

**The goal:** Maximize cost efficiency without sacrificing availability.

---

## 🏗️ Architecture Overview

At a high level, this setup looks like:

![Adaptive HTTP]({{ site.baseurl }}/images/2025-05-27/adaptive-http.png)

A separate Lambda controller function monitors traffic (via CloudWatch alarms) and adjusts the system accordingly.

---

## ⚙️ The Code: One App, Two Runtimes

You can run the same Express.js app on both ECS and Lambda with minimal changes.

### In ECS

You deploy it as a typical containerized app on Fargate.

### In Lambda

You wrap the Express app using a library like `serverless-http`:

```
    const express = require('express');
    const serverless = require('serverless-http');

    const app = express();
    // define routes here

    exports.handler = serverless(app);
```

---

## 🔀 Load Balancer Setup with Weighted Target Groups

Your ALB listener forwards traffic to a **forward action** with two target groups:

- ECS Target Group (type: `ip`)
- Lambda Target Group (type: `lambda`)

You assign weights to these target groups. Initially:

- ECS weight = 100
- Lambda weight = 0

> **Important!** -
> If ECS has zero healthy targets, traffic will *still* route to Lambda—even with a weight of 0. This provides seamless fallback during ECS spin-up.

---

## 📉 Responding to Low Traffic

You'll need to create an alarm to respond to the changes in traffic in ECS. You can configure the alarm to whatever levels makes sense for you. The alarm should be in an OK state when traffic is high enough to justify using ECS, and in an ALARM state when it should be switched to Lambda.

> **Tip** -
> I like to use EventBridge to trigger a Lambda when the state changes, but you can also connect to the alarm directly.

When a CloudWatch alarm detects low traffic:

1. A Lambda controller function is triggered.
2. It updates the ALB listener rule to:
   - Set ECS weight to 0
   - Set Lambda weight to 100
3. It scales down the ECS service to 0 tasks.

This stops container usage completely, minimizing costs while keeping the service available via Lambda.

---

## 📈 Responding to High Traffic

On rising demand:

1. The same CloudWatch alarm triggers the controller Lambda.
2. It:
   - Sets ECS weight to 100
   - Sets Lambda weight to 0
3. It scales up the ECS service (e.g., to your default task count).

While the ECS service is starting, if no healthy targets are available, the ALB continues routing traffic to Lambda—even with weight 0—ensuring a smooth transition.

---

## ✅ Benefits

- **Efficiency:** Save money by not running idle ECS tasks.
- **Resilience:** Lambda catches any gaps during ECS startup.
- **Simplicity:** One codebase, two runtimes.
- **Flexibility:** Control via Lambda and CloudWatch means no manual intervention.

---

## ⚠️ Considerations

- **Authentication and authorization** If you're a serverless person you are probably used to using some authorization at the point of entry (e.g., at an API Gateway or AppSync). With this model you no longer have the zero trust model of IAM. You can use network security to be sure only certain sources can access your service, just be aware of the implications of doing so.
- **Cold Starts:** Lambda functions might introduce latency during first invocation.
- **Startup Time:** ECS services take time to start; make sure your fallback duration is appropriate.

---

## 🧠 Final Thoughts

This adaptive pattern gives you the best of both worlds: the scalability and efficiency of ECS for peak loads, and the cost savings and simplicity of Lambda when traffic drops. It has some limitations, and it's not the right solution for everyone, but it does start to look a little like that future Allen was talking about.

Check out a working example of this [here](https://github.com/jasonwadsworth/blog-code/tree/main/adaptive-http)