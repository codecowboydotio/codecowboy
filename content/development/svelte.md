+++
title = "A Simple svelte app"
date = "2025-06-19"
aliases = ["dev"]
tags = ["dev"]
categories = ["software", "dev"]
author = "codecowboy.io"
+++

# Intro
I recently had a need to think about API testing again. I needed to quickly come up with a small web based tester to perform the occasional get request but also a post request with data. 

It's a lot simpler for me to just grab a JSON blob of data and send it to an endpoint with a click and evaluate the results. 

I decided to use svelte. 

I haven't used svelte before but I was surprised by its simplicity and ease of use.
I also admit that I like the fact it has easy templating. :)

# TLDR
If you just want the code - it's here:

[https://github.com/codecowboydotio/mcp-server-examples](https://github.com/codecowboydotio/mcp-server-examples)


# High level overview
My app does two things:

1. It can perform a GET request and display the result
2. It can perform a POST request and display the result

I find this very useful when I am developing services that are abstracted away in AWS, or some other remote framework and I want to check to see if something is working end to end during the development process.

# Install the environment
First I need to create a project directory and then run through some generic project starter questions.

I use **npx** to create the initial project structure and scaffold code.

```Shell
npx sv create my-svelte-app
Need to install the following packages:
sv@0.8.11
Ok to proceed? (y) y

┌  Welcome to the Svelte CLI! (v0.8.11)
│
◇  Which template would you like?
│  SvelteKit minimal
│
◇  Add type checking with TypeScript?
│  Yes, using TypeScript syntax
│
◆  Project created
│
◇  What would you like to add to your project? (use arrow keys / space bar)
│  devtools-json
│
◆  Successfully setup add-ons
│
◇  Which package manager do you want to install dependencies with?
│  npm
│
◆  Successfully installed dependencies
│
◇  Project next steps ─────────────────────────────────────────────────────╮
│                                                                          │
│  1: cd my-svelte-app                                                     │
│  2: git init && git add -A && git commit -m "Initial commit" (optional)  │
│  3: npm run dev -- --open                                                │
│                                                                          │
│  To close the dev server, hit Ctrl-C                                     │
│                                                                          │
│  Stuck? Visit us at https://svelte.dev/chat                              │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────╯
│
└  You're all set!

```

# Project Structure
The project structure looks something like this:

I am using a single file layout for my svelte app. This is because I personally find it simpler to do this, and because it's a small app.

You will note though that there is provision in the project structure to break out components such as CSS and so on.

```Shell
api-client/
├── src/
│   ├── app.css                 # Global CSS styles
│   ├── app.html                # Main HTML template
│   ├── lib/
│   └── routes/
│       ├── +page.svelte        # Main API client page
├── static/
├── tests/
├── .gitignore
├── package.json
├── README.md
├── svelte.config.js           # SvelteKit configuration
├── tsconfig.json              # TypeScript config (if using TS)
└── vite.config.js             # Vite configuration
```


# The code
As I am using a single file, I will walk through the different components.

## Imports
The two imports at the top essentially check if the app is running in a browser or not.

This is described in the svelte docs here:
[https://svelte.dev/docs/kit/$app-environment](https://svelte.dev/docs/kit/$app-environment)

After the imports, I set some defaults that are used within the app. 

```Javascript
<!-- src/routes/+page.svelte -->
<script>
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let url = '';
  let requestData = '{\n  "key": "value"\n}';
  let response = '';
  let loading = false;
  let error = null;
  let showResult = false;
  let method = 'POST'; // Default method selection
```

## Handle form submission
The handle form submission function does most of the heavy lifting within the app.

As the app can handle both GET and POST requests, the first try / catch block checks the method to see if it is a POST. If the method that is received is a POST then the incoming JSON data is parsed for correctness.

```Javascript
  async function handleSubmit() {
    if (!browser) return;
    
    loading = true;
    error = null;
    showResult = true;

    try {
      // Validate URL
      if (!url) throw new Error('URL is required');

      // For POST requests, validate JSON
      let parsedData;
      if (method === 'POST') {
        try {
          parsedData = JSON.parse(requestData);
        } catch (e) {
          throw new Error(`Invalid JSON: ${e.message}`);
        }
      }
```

The next statements set the content type, and also include the data portion of the request - only if this is a POST request.

```Javascript
      // Configure fetch options based on the selected method
      const fetchOptions = {
        method: method,
        headers: {
          'Content-Type': 'application/json'
        }
      };

      // Only include body for POST requests
      if (method === 'POST') {
        fetchOptions.body = requestData;
      }
```

Then make the request:

```Javascript
const fetchResponse = await fetch(url, fetchOptions);
```

Finally, once the request has returned, be it a GET or a POST, check the headers, and return the response as JSON. If the response is returned not as JSON explcitly, then perform an explicit fetch as json.

```Javascript
      const fetchResponse = await fetch(url, fetchOptions);

      const contentType = fetchResponse.headers.get('content-type');
      if (contentType && contentType.includes('application/json')) {
        const jsonResponse = await fetchResponse.json();
        return JSON.stringify(jsonResponse, null, 2);
      } else {
        const textResponse = await fetchResponse.text();
        return textResponse;
      }
    } catch (err) {
      error = err.message;
      return `Error: ${err.message}`;
    } finally {
      loading = false;
    }
  }

```

## Make the request
The make request function just calls the handle submit function.
The heavy lifting is all done by the other function.

```Javascript
  async function makeRequest() {
    response = await handleSubmit();
  }
```

## Format the json and close the popup box
The next two functions just format the JSON data, and close the object that shows the result by setting it to false.

```Javascript
  function formatJson() {
    try {
      const parsed = JSON.parse(requestData);
      requestData = JSON.stringify(parsed, null, 2);
    } catch (e) {
      // Ignore formatting if JSON is invalid
    }
  }

  function closeResult() {
    showResult = false;
  }
</script>
```

# The HTML
As svelte is a templating language, the script above does the work, but within the HTML and CSS there are sections that both make the request but also handle the result.

Svelte is sort of cool in the way that it handles this. It's a reactive framework 


### Form Design
The goal of designing the form was to make it easy to use. The form changes based on the selected HTTP method. The URL input is always visible, but the JSON body textarea only appears for POST requests:

```svelte
{#if method === 'POST'}
  <div class="form-group">
    <label for="data">
      Request Body (JSON)
      <button type="button" class="format-btn" on:click={formatJson}>Format</button>
    </label>
    <textarea
      id="data"
      bind:value={requestData}
      placeholder='"key": "value"'
      rows="10"
      required
    ></textarea>
  </div>
{/if}
```

The "Format" button will automatically prettify JSON input – a small touch that makes the tool easier to use.

### The Popup Response Window

Instead of a traditional sidebar or inline display, the response appears in a modal popup. This makes the response easy to read, and also means that you need to take action before hitting the request button again.

The popup structure is straightforward:

```svelte
{#if showResult}
  <div class="popup-overlay" on:click={closeResult}>
    <div class="popup-window" on:click|stopPropagation>
      <div class="popup-header">
        <h2>API Response</h2>
        <button class="close-btn" on:click={closeResult}>×</button>
      </div>
      <!-- Response content here -->
    </div>
  </div>
{/if}
```

I used `on:click|stopPropagation` on the popup window to prevent it from closing when users click inside it, but clicking the overlay dismisses the popup – a UX pattern most users expect.

## Loading States and Error Handling

There are three distinct states for the response area:

1. **Loading**: Shows a spinning animation while the request is in progress
2. **Error**: Displays error messages in a red-bordered container
3. **Success**: Shows the formatted response in a code block

```svelte
{#if loading}
  <div class="loading">
    <div class="spinner"></div>
    <p>Sending request...</p>
  </div>
{:else if error}
  <div class="error">
    <p>{error}</p>
  </div>
{:else}
  <pre>{response}</pre>
{/if}
```

The loading spinner uses a simple CSS animation. This animation just spins. :)

```css
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```

## How to Use It

Using my API client is straightforward:

1. **Enter the API endpoint URL** in the first field
2. **Select GET or POST** from the dropdown
3. **For POST requests**, add your JSON payload in the textarea (use the Format button to prettify it)
4. **Click "Send Request"** and watch the response appear in the popup

The tool handles the rest – parsing responses, formatting JSON, showing loading states, and displaying errors clearly.

![UI](/images/svelte-1.jpg)


![spinner](/images/svelte-2.jpg)


![response](/images/svelte-3.jpg)


## Complete Source Code

Here's the full SvelteKit component that powers the API client:

```svelte
<!-- src/routes/+page.svelte -->
<script>
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let url = '';
  let requestData = '{\n  "key": "value"\n}';
  let response = '';
  let loading = false;
  let error = null;
  let showResult = false;
  let method = 'POST';

  async function handleSubmit() {
    if (!browser) return;
    
    loading = true;
    error = null;
    showResult = true;

    try {
      if (!url) throw new Error('URL is required');

      let parsedData;
      if (method === 'POST') {
        try {
          parsedData = JSON.parse(requestData);
        } catch (e) {
          throw new Error(`Invalid JSON: ${e.message}`);
        }
      }

      const fetchOptions = {
        method: method,
        headers: {
          'Content-Type': 'application/json'
        }
      };

      if (method === 'POST') {
        fetchOptions.body = requestData;
      }

      const fetchResponse = await fetch(url, fetchOptions);

      const contentType = fetchResponse.headers.get('content-type');
      if (contentType && contentType.includes('application/json')) {
        const jsonResponse = await fetchResponse.json();
        return JSON.stringify(jsonResponse, null, 2);
      } else {
        const textResponse = await fetchResponse.text();
        return textResponse;
      }
    } catch (err) {
      error = err.message;
      return `Error: ${err.message}`;
    } finally {
      loading = false;
    }
  }

  async function makeRequest() {
    response = await handleSubmit();
  }

  function formatJson() {
    try {
      const parsed = JSON.parse(requestData);
      requestData = JSON.stringify(parsed, null, 2);
    } catch (e) {
      // Ignore formatting if JSON is invalid
    }
  }

  function closeResult() {
    showResult = false;
  }
</script>

<svelte:head>
  <title>API Client - SvelteKit</title>
  <meta name="description" content="Simple API client for testing REST endpoints" />
</svelte:head>

<main>
  <div class="container">
    <div class="input-section">
      <h1>API Client</h1>

      <form on:submit|preventDefault={makeRequest}>
        <div class="form-group">
          <label for="url">API Endpoint URL</label>
          <input
            type="text"
            id="url"
            bind:value={url}
            placeholder="https://api.example.com/endpoint"
            required
          />
        </div>

        <div class="form-group">
          <label for="method">Request Method</label>
          <select id="method" bind:value={method}>
            <option value="GET">GET</option>
            <option value="POST">POST</option>
          </select>
        </div>

        {#if method === 'POST'}
          <div class="form-group">
            <label for="data">
              Request Body (JSON)
              <button type="button" class="format-btn" on:click={formatJson}>Format</button>
            </label>
            <textarea
              id="data"
              bind:value={requestData}
              placeholder='"key": "value"'
              rows="10"
              required
            ></textarea>
          </div>
        {/if}

        <button type="submit" class="submit-btn" disabled={loading}>
          {loading ? 'Sending Request...' : `Send ${method} Request`}
        </button>
      </form>
    </div>

    {#if showResult}
      <div class="popup-overlay" on:click={closeResult}>
        <div class="popup-window" on:click|stopPropagation>
          <div class="popup-header">
            <h2>API Response</h2>
            <button class="close-btn" on:click={closeResult}>×</button>
          </div>

          <div class="popup-content">
            {#if loading}
              <div class="loading">
                <div class="spinner"></div>
                <p>Sending request...</p>
              </div>
            {:else if error}
              <div class="error">
                <p>{error}</p>
              </div>
            {:else}
              <pre>{response}</pre>
            {/if}
          </div>
        </div>
      </div>
    {/if}
  </div>
</main>

<style>
  /* Global styles and component styling */
  :global(body) {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
      Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    margin: 0;
    padding: 0;
    background-color: #f7f9fc;
    color: #333;
  }

  .container {
    min-height: 100vh;
    max-width: 800px;
    margin: 0 auto;
    padding: 2rem;
  }

  .input-section {
    width: 100%;
  }

  h1 {
    color: #2d3748;
    margin-bottom: 1.5rem;
  }

  .form-group {
    margin-bottom: 1.5rem;
  }

  label {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-weight: 600;
    margin-bottom: 0.5rem;
    color: #4a5568;
  }

  input, textarea, select {
    width: 100%;
    padding: 0.75rem;
    border: 1px solid #e2e8f0;
    border-radius: 0.25rem;
    font-size: 1rem;
    font-family: inherit;
    box-sizing: border-box;
  }

  textarea {
    font-family: monospace;
    resize: vertical;
  }

  .format-btn {
    background: none;
    border: none;
    color: #4299e1;
    cursor: pointer;
    font-size: 0.875rem;
    padding: 0.25rem 0.5rem;
  }

  .submit-btn {
    background-color: #4299e1;
    color: white;
    border: none;
    border-radius: 0.25rem;
    padding: 0.75rem 1.5rem;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.2s;
  }

  .popup-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }

  .popup-window {
    background-color: white;
    border-radius: 0.5rem;
    box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
    max-width: 90vw;
    max-height: 80vh;
    width: 700px;
    display: flex;
    flex-direction: column;
  }

  .spinner {
    border: 3px solid #e2e8f0;
    border-top: 3px solid #4299e1;
    border-radius: 50%;
    width: 30px;
    height: 30px;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
</style>
```


# Conclusion
