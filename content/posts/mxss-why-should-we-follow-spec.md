---
title: "MXSS: Why Should We Follow Spec"
date: 2024-08-03T13:54:16+05:30
draft: false
---

Edit: freddyb from Firefox reminded me about RETURN_DOM_FRAGMENT flag in DOMPurify which effectiely removes second parsing and the mxss problem too, read about it here

https://x.com/freddyb/status/1843916184477126945

# Why should we follow the spec?

## Following the Spec Might Not Always Ensure Security?

**Parsing Differentials**

HTML sanitization is a challenging problem because the HTML needs to be parsed during sanitization to remove potentially dangerous elements. However, when the sanitized HTML string is assigned to the browser DOM, it gets parsed again.

The problem with this approach is that parsing the same HTML string multiple times may not produce the same result. This behavior can be called non-idempotent

\\( P(P(H)) \neq P(H)\\)

Where:

  * H is the original HTML string that needs to be sanitized.

  * P(H) represents the parsing function applied to H.




However, the issue is not just with one parse; each time the HTML is parsed, you can get different results. So, after parsing H n times, we get n different results. This can be expressed as:

\\(P^n(H) \neq P^{n-1}(H) \neq P^{n-2}(H) ... \\)

The code representation of non-idempotent HTML parsing looks like this:
    
    
    H = '<form><div></form><form></div></form>'var P_H = new DOMParser().parseFromString(H, "text/html")var P_H_P_H = new DOMParser().parseFromString(P_H.documentElement.outerHTML, "text/html")

To see an example of the non-idempotent behavior, you can run this code in the browser's console. You will observe the below

\\( H = \texttt{<form><div></form><form></div></form>}\\)

\\( P(H) = \texttt{<form><div><form></form></div></form>}\\)

\\( P(P(H)) = \texttt{<form><div></div></form>}\\)

You will observe that the second parsing (P_H_P_H) produces a different HTML structure compared to the first parsing (P_H), demonstrating the mutation. And, In the HTML specification, this behavior comes with a big warning:

[![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb4fd0d92-481e-42c6-ab77-994bb401efd5_2692x234.png)](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb4fd0d92-481e-42c6-ab77-994bb401efd5_2692x234.png)

And as per the warning, this **non-idempotency** of HTML parsing has led to numerous client-side HTML sanitizer bypasses. List of these bypasses can be found [here](https://sonarsource.github.io/mxss-cheatsheet/examples/).

 **Google MXSS**

A classic example of non-idempotent HTML parsing can be seen in the infamous Google Search MXSS exploit. In this case, the HTML sanitizer encounters a string where a `</noscript>` tag is embedded inside an attribute:
    
    
    <noscript><p title="</noscript><img src=x onerror=alert(1)>">

During the first parsing, the sanitizer doesn’t recognize the _< /noscript>_ tag as a closing element because the behavior of _< noscript>_ [varies depending on whether JavaScript is enabled or not](https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-inhead:~:text=name%20is%20%22noembed%22-,A%20start%20tag%20whose%20tag%20name%20is%20%22noscript%22%2C%20if%20the%20scripting%20flag%20is%20enabled,-Follow%20the%20generic). In this case, with JavaScript enabled, the _< /noscript>_ tag inside the attribute is treated as part of the attribute's value and the leaving the _< noscript>_ content appearing harmless.

However, when the sanitized string is reassigned to an _iframe_ using the _srcdoc_ attribute, the browser parses the content again. This time, the _< /noscript>_ tag is properly closed, causing the content after it (like the _< img>_ tag with the _onerror_ event) to be rendered. see below image.

[![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F16a42da3-eca6-4f01-a725-8c2b34908217_1958x1356.png)](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F16a42da3-eca6-4f01-a725-8c2b34908217_1958x1356.png)

 **Why should we follow spec if it leads to security issues?**

Now coming to the main topic of this blog: Why should we follow the spec?

The primary reason client-side sanitizers are often used instead of server-side sanitizers is due to parser differentials. What happens if the server-side sanitizer deems the HTML safe, but when parsed by the client, it becomes dangerous? This discrepancy can lead to security vulnerabilities.

Interestingly, the same issue affects client-side sanitizers, as seen with the non-idempotency discussed earlier. When the same HTML string is parsed multiple times, it can yield different results, leading to potential security bypasses.

Now I wonder: **What if we make parsing idempotent** , such that:

\\( P(P(H)) = P(H)\\)

This means the sanitizer would work on the same parsed DOM tree as the one rendered by the browser, ensuring that subsequent parsing does not introduce any unexpected mutations. Wouldn't this approach effectively fix **MXSS** ?

To achieve this idempotency, we might need to stray from the standard specification by writing a custom parser, not one that strictly complies with the HTML spec, but rather one that ensures the parsed tree remains consistent—i.e., the output of parsing the HTML once and then parsing it again should yield the same tree. The custom parser could then iterate over this tree and safely remove any dangerous elements. 

As shown below, where P is the browser’s parser and Px​ is our custom parser, our goal is to make parsing **idempotent** by ensuring that after the browser parses the sanitized string produced by our custom parser, the result remains unchanged.

\\( P(P_x(H)) = P_x(H)\\)

I understand this idea could introduce implementation issues, possibly leading to bypasses in edge cases. However, as of now, I don't see the same types of mutations happening with this approach. Am I wrong?????

