# Frontend Monitoring: Because Backends Aren't the Only Things That Break

## Integrating New Relic with Next.js

This section outlines the steps to integrate New Relic's Browser agent into your Next.js application for frontend monitoring.
The emojis are used sparingly and in places where they add a touch of humor or a visual cue without being overwhelming. The humor is subtle and relevant to the developer experience. Remember to replace `"YOUR_INSERT_API_KEY"` with your actual API key.

**1. Sign Up and Obtain Your Insert API Key:**

- If you don't already have a New Relic account, sign up at [https://newrelic.com/signup](https://newrelic.com/signup).
- Create a new application in New Relic.
- Navigate to the "Browser" section of your application's settings.
- Locate your Insert API key. **Keep this key secure!** It's crucial for the integration.

**2. Install the New Relic Browser Agent:**

You don't need to install a separate package for the New Relic Browser agent. The agent is injected via a script tag.

**3. Add the New Relic Script Tag:**

The simplest way to add the New Relic Browser agent is to include the following script tag within the `<Head>` component of your Next.js application's pages. This ensures the agent loads on every page.

```javascript
// pages/_app.js or pages/_document.js (depending on your Next.js version)

import { useEffect } from 'react';
import Script from 'next/script'; // Import the Script component

function MyApp({ Component, pageProps }) {
  useEffect(() => {
    // Optionally, you can add logic here to conditionally load the script
    // based on environment variables or other factors.
  }, []);

  return (
    <>
      <Script
        id="newrelic-script"
        strategy="afterInteractive"
        src={`https://js-agent.newrelic.com/nr-agent.min.js?license=YOUR_INSERT_API_KEY`}
      />
      <Component {...pageProps} />
    </>
  );
}

export default MyApp;

This guide provides a step-by-step walkthrough for integrating New Relic into your frontend application. We'll cover the essential steps to ensure comprehensive monitoring and performance tracking, enabling you to quickly identify and resolve issues impacting the user experience. This guide assumes familiarity with basic frontend development concepts and Next.js framework. Let's get started!
```
