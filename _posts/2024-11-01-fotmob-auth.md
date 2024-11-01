---
layout: post
title: "Reverse Engineering Fotmob's API Authentication"
categories: security
---

## Introduction
As the creator of a fantasy football site, I rely on Fotmob's API to fetch crucial XG (Expected Goals) and XA (Expected Assists) data. Recently, I encountered an unexpected hurdle: my API calls started receiving `Unauthorized` status codes. This sudden change in API behavior set me on an intriguing journey of reverse engineering and problem-solving.

## 1. The Challenge
My fantasy football site's API calls to Fotmob suddenly started returning Unauthorized status codes.
I realized that Fotmob's own frontend was still able to access the data successfully.
This led me to investigate how their frontend was making these API calls.

### 1.1. The Investigation
I turned to Chrome's network tab to analyze the requests made by Fotmob's frontend.
I discovered a new header in their requests: "X-Fm-Req".
This header seemed to be the key to authenticating requests to their API.

![Screenshot of Chrome's network tab showing the "X-Fm-Req" header](/images/fotmob_req_header.png)

### 1.2. Encode the X-Fm-Req token
I asked AI Perplexity if it could help me decipher this token, and directly it responded with:

`The token you've shared is actually a Base64-encoded JSON object, not a standard JWT. Let's decode it and examine its structure:`

```json
{
  "body": {
    "url": "/api/leagueseasondeepstats?id=67&season=22583&type=players&stat=expected_goals",
    "code": 1730380499852
  },
  "signature": "610A1D52E15A31C70DDA2B19E6DEF79D"
}
```

This got me really excited! 

### 1.3. The Deep Dive
To understand how this "X-Fm-Req" header was generated, I needed to dig deeper:
- I started analyzing the JavaScript code responsible for making API calls on Fotmob's site.
- The relevant code was minified and obfuscated, making it challenging to decipher.
- I focused on identifying the specific snippets responsible for generating the header.

![Signature found](/images/fotmob_signature_found.png)

Here's a segment of the first i found:

```js
...

, g = (e, t) => u(`${JSON.stringify(e)}${t}`)
, m = function(e) {
let t = arguments.length > 1 && void 0 !== arguments[1] ? arguments[1] : new Date
    , r = {
    url: e,
    code: t.getTime()
}
    , n = g(r, h);
return btoa(JSON.stringify({
    body: r,
    signature: n
}))
```

## 2. The Solution: Unraveling the Authentication Puzzle
After careful analysis and numerous iterations, I finally cracked the code behind Fotmob's authentication mechanism. 
Here's how the pieces came together:

### 2.1. Discovering the Inputs
Through careful examination of the network requests in Chrome's DevTools, I identified the key components used to generate the "X-Fm-Req" header:

1. **API Endpoint URL**: The specific path being requested.
   Example: `/api/leagueseasondeepstats?id=67&season=22583&type=players&stat=expected_goals`

2. **Timestamp**: A Unix timestamp in milliseconds.
   Example: `1730380499852`

3. **Secret String**: This was the trickiest part to uncover. After deobfuscating the JavaScript, I discovered it was actually song lyrics!

### 2.2. The Authentication Recipe
The "X-Fm-Req" header is generated using the following steps:

1. Create a JSON object with the URL and timestamp:
```js
{
    url: "/api/leagueseasondeepstats?id=67&season=22583&type=players&stat=expected_goals",
    code: 1730380499852
}
```
Stringify this JSON object.

2. Concatenate the stringified JSON with the secret lyrics:
```js
const h = `We're no strangers to love
You know the rules and so do I
A full commitment's what I'm thinking of
You wouldn't get this from any other guy
...`
```
Generate an MD5 hash of this concatenated string.
Convert the hash to uppercase.

I created a script in VS Code and debugged it to try to get the same output.
After some time with a lot of help from Perplexity I got it to work!

Now the harder part begun: Trying to implement the exact same logic in C#.

### 2.3. The Challenge of Implementation
Translating this process from JavaScript to C# presented several challenges:
- *MD5 Implementation*: Ensuring the C# MD5 function produced identical results to the JavaScript version.
- *String Handling*: Managing differences in how C# and JavaScript handle string encodings and line endings.
- *JSON Serialization*: Matching the exact JSON string output from JavaScript in C#.

### 2.4. Verifying the Solution
To confirm the correct implementation, I used known inputs to produce a known output:

Inputs:
- URL: /api/leagueseasondeepstats?id=67&season=22583&type=players&stat=expected_goals
- Timestamp: 1730380499852
- Secret Lyrics

Expected Output: `610A1D52E15A31C70DDA2B19E6DEF79D`

The last piece that solved the puzzle was when we converted the JS input string and C# input string to hex values.
Then we found out that the line endings where different.

After multiple refinements, my C# implementation finally produced this exact output, confirming that I had successfully reverse-engineered the authentication mechanism.

### 2.5. Key Takeaways
- Patience and persistence were crucial in deciphering the obfuscated JavaScript.
- Understanding the nuances of cross-language implementation was vital for accurate replication.
- This process highlighted the importance of robust API security measures and the challenges in balancing security with usability.

### 2.6. Complete C# solution for creating the token

```cs
private static readonly string SecretLyrics = 
@"We're no strangers to love
You know the rules and so do I
A full commitment's what I'm thinking of
You wouldn't get this from any other guy
...";

public static string GenerateToken(string url)
{
    var code = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();

    var body = new
    {
        url,
        code
    };

    var signature = GetSignature(body);

    var token = new
    {
        body,
        signature
    };

    return Convert.ToBase64String(Encoding.UTF8.GetBytes(JsonSerializer.Serialize(token)));
}

private static string GetSignature(object body)
{
    string jsonBody = JsonSerializer.Serialize(body, new JsonSerializerOptions
    {
        Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping,
        WriteIndented = false                 
    });

    string combinedString = (jsonBody + SecretLyrics.Replace("\r\n", "\n")).TrimEnd('\n');

    return GenerateMd5Hash(combinedString);            
}

private static string GenerateMd5Hash(string input)
{
    using (var md5 = MD5.Create())
    {
        byte[] inputBytes = Encoding.UTF8.GetBytes(input);
        byte[] hashBytes = md5.ComputeHash(inputBytes);

        var sb = new StringBuilder();
        for (int i = 0; i < hashBytes.Length; i++)
        {
            sb.Append(hashBytes[i].ToString("x2"));
        }
        return sb.ToString().ToUpper();
    }
}
```

## 7. Ethical Considerations
While reverse engineering can be an exciting challenge, it's crucial to approach it ethically and responsibly:

- The purpose of this exercise was purely educational and to maintain the functionality of my fantasy football site.
- Before publishing any findings, I reached out to Fotmob to inform them about the discovery.
- It's important to note that this authentication method likely isn't a critical security measure, but rather a way to prevent casual scraping.
- Throughout this process, no harm was done to Fotmob's APIs or services.

Remember, the goal of such exercises should be learning and improving our understanding of web security, not exploiting or compromising others' systems.

## 8. Lessons Learned
This journey taught me valuable lessons:
- The importance of understanding web security from both offensive and defensive perspectives.
- The intricacies of cross-language implementation, especially in cryptographic functions.
- The value of persistence and methodical debugging in solving complex problems.

## 9. Conclusion
Reverse engineering Fotmob's API authentication was a challenging but rewarding experience. It highlighted the complexities of web security and the importance of thorough analysis when dealing with API integrations. As web developers, we should always strive to understand the systems we interact with, while maintaining ethical standards and respecting the work of others.
Remember, the goal of such exercises should always be learning and improving our own systems, not exploiting or compromising others. Happy coding, and may your API calls always return 200 OK! 💪

![Never gonna!](/images/fotmob_rick.webp)