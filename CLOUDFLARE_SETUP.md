# How to Use Cloudflare Workers to Secure Your DeepSeek API Key

This guide will show you how to create a Cloudflare Worker that acts as a secure proxy for your DeepSeek API, keeping your API key hidden from the client-side code.

## Step 1: Create a Cloudflare Account

1. Go to [cloudflare.com](https://cloudflare.com) and sign up for a free account
2. Verify your email address

## Step 2: Create a Cloudflare Worker

1. Log in to your Cloudflare dashboard
2. Click on **Workers & Pages** in the left sidebar
3. Click **Create application**
4. Select **Create Worker**
5. Give your worker a name (e.g., `deepseek-proxy`)
6. Click **Deploy**

## Step 3: Write the Worker Code

Replace the default code in the worker editor with this:

```javascript
export default {
  async fetch(request, env, ctx) {
    // Only allow POST requests
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    // Optional: Add CORS headers if needed
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*', // Change to your domain for production
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    // Handle preflight requests
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    try {
      // Get the request body
      const body = await request.json();

      // Forward the request to DeepSeek API
      const response = await fetch('https://api.deepseek.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${env.DEEPSEEK_API_KEY}` // API key from environment variable
        },
        body: JSON.stringify(body)
      });

      // Get the response from DeepSeek
      const data = await response.json();

      // Return the response with CORS headers
      return new Response(JSON.stringify(data), {
        headers: {
          'Content-Type': 'application/json',
          ...corsHeaders
        }
      });
    } catch (error) {
      return new Response(JSON.stringify({ error: error.message }), {
        status: 500,
        headers: {
          'Content-Type': 'application/json',
          ...corsHeaders
        }
      });
    }
  }
};
```

## Step 4: Set Up Environment Variables (Secret Key)

1. In your Worker editor, click on **Settings** tab
2. Scroll down to **Variables**
3. Click **Add variable**
4. Variable name: `DEEPSEEK_API_KEY`
5. Variable value: Enter your DeepSeek API key (`sk-0fdaddd9f0db4e70821938381de23af6`)
6. **Important**: Make sure to select **Encrypt** or mark it as a **Secret** (this keeps it hidden)
7. Click **Save**

## Step 5: Deploy the Worker

1. Click **Save and Deploy** in the worker editor
2. Your worker will be deployed and you'll get a URL like:
   `https://deepseek-proxy.your-subdomain.workers.dev`

## Step 6: Update Your HTML Files

1. Open `index.html` and `index2.html`
2. Find the line:
   ```javascript
   const CLOUDFLARE_WORKER_URL = 'https://your-worker.your-subdomain.workers.dev/api/deepseek';
   ```
3. Replace it with your actual worker URL:
   ```javascript
   const CLOUDFLARE_WORKER_URL = 'https://deepseek-proxy.your-subdomain.workers.dev';
   ```

## Step 7: Test Your Setup

1. Open your HTML file in a browser
2. Try using the worksheet generator
3. Check the browser console for any errors
4. If you see CORS errors, make sure your CORS headers in the worker match your domain

## Optional: Add Rate Limiting or Authentication

You can enhance security by adding:

### Rate Limiting
```javascript
// Add at the top of your worker
const RATE_LIMIT = 10; // requests per minute
const rateLimitMap = new Map();

// Add before the fetch call
const clientIP = request.headers.get('CF-Connecting-IP') || 'unknown';
const now = Date.now();
const windowStart = now - 60000; // 1 minute window

if (!rateLimitMap.has(clientIP)) {
  rateLimitMap.set(clientIP, []);
}

const requests = rateLimitMap.get(clientIP).filter(time => time > windowStart);
if (requests.length >= RATE_LIMIT) {
  return new Response('Rate limit exceeded', { status: 429 });
}

rateLimitMap.get(clientIP).push(now);
```

### API Key Authentication (Optional)
```javascript
// Check for a secret key in the request
const providedKey = request.headers.get('X-API-Key');
if (providedKey !== env.SECRET_KEY) {
  return new Response('Unauthorized', { status: 401 });
}
```

Then in your HTML, add:
```javascript
headers: {
  'Content-Type': 'application/json',
  'X-API-Key': 'your-secret-key-here'
}
```

## Troubleshooting

### CORS Errors
- Make sure your worker returns CORS headers
- Update `Access-Control-Allow-Origin` to your specific domain instead of `*`

### 401 Unauthorized
- Check that your `DEEPSEEK_API_KEY` environment variable is set correctly
- Make sure it's marked as encrypted/secret

### 500 Internal Server Error
- Check the Cloudflare Worker logs in the dashboard
- Verify your DeepSeek API key is valid

## Benefits of This Approach

1. **Security**: Your API key is never exposed to the client
2. **Control**: You can add rate limiting, logging, and other controls
3. **Free**: Cloudflare Workers free tier includes 100,000 requests per day
4. **Fast**: Workers run on Cloudflare's edge network worldwide

## Additional Resources

- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Cloudflare Workers Examples](https://developers.cloudflare.com/workers/examples/)
- [Environment Variables in Workers](https://developers.cloudflare.com/workers/configuration/environment-variables/)
