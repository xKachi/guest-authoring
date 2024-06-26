---
title: "Building a URL Shortener with React, TypeScript, and Directus"
description: "Learn to build a link shortener using Directus and React! Define data models, customize roles, and dynamically route with ease."
author:
  name: "Onyedikachi Eni"
  avatar_file_name: "profile-photo.jpg"
---

In this tutorial, we will build a link shortener. Then, we'll create a React project that looks for a slug, queries the associated record in Directus, and redirects the user to the configured link.

## Before You Start

You will need:

1. A Directus project - follow our [quickstart guide](https://docs.directus.io/getting-started/quickstart) if you don't already have one.
2. Some knowledge of Javascript and React.js.

The complete code for this project can be found on [GitHub](https://github.com/xKachi/Link-Shortener/tree/main)

## Setting Up Your Directus Project

Create a new collection called `short_link`. Enable all optional fields, and add the following additional fields:

- `slug` (type: String, interface: input, required): The URL path that will be shared in the short URL.
- `url` (type: String, interface: input, required): The URL we want to redirect to.
- `clicks` (type: Integer, interface: input, default_value: 0): Number of times the link was clicked.

To make sure the URL that will be stored in the **URL** field is valid, we will use a validation filter.

Create a validation rule for the URL. From the field settings, ensure the URL **Matches RegExp**:

```
https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)
```

![Validation regex](<Setup a Directus Collection_3.PNG>)
Create a new `contributor` role will have the following privileges:

- Create a short_link.
- Read a short_link
- Update the `clicks` field (to avoid spamming) of short_link.

![Contributor Permissions - create and read short link are enabled. Edit is custom.](<Customizing Roles and Permissions_2.PNG>)

![Contributor Privileges](<Customizing Roles and Permissions_3.PNG>)

Finally, create some sample data in your `short_link` collection. 

| Slug       | URL                    |
| ---------- | ---------------------- |
| website    | https://directus.io/   |
| x          | https://x.com/directus |

## Setting Up Your React Project

Install React.js, set up dependencies, and run a development server:

```
npm create vite@latest link-shortener
? Select a framework: React
? Select a variant: TypeScript

npm install
npm install --save-dev @types/node
npm install react-router-dom
npm install @directus/sdk 

npm run dev
```

Now, open http://localhost:5173/ on your browser, and you should see the starter page.

Next, we will set up Vite to allow environment variables, which will be used to store our Directus credentials. In the `vite.config.ts` file, add the following code:

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  define: {
    "process.env.VITE_DIRECTUS_API_TOKEN.": JSON.stringify(
      process.env.VITE_DIRECTUS_API_TOKEN
    ),
    "process.env.VITE_DIRECTUS_API_URL.": JSON.stringify(
      process.env.VITE_DIRECTUS_API_URL
    ),
  },
});
```

Create a `.env` file in the `src` directory, and the following variables, being sure to provide your specific static authentication token and project URL:

```
VITE_DIRECTUS_API_TOKEN = XXXXX
VITE__DIRECTUS_API_URL = https://your-amazing.directus.app/
```

In the `src` directory create a sub-directory called `utils`, and within it, a `directus.ts` file.

Add the following code to _directus.ts_:

```typescript
import { createDirectus, staticToken, rest } from "@directus/sdk";

const directusToken = import.meta.env.VITE_DIRECTUS_API_TOKEN;
const directusUrl = import.meta.env.VITE__DIRECTUS_API_URL;

if (!directusToken) {
  throw new Error("Please include a Token");
}

if (!directusUrl) {
  throw new Error("Please include a Url");
}

export const directus = createDirectus(directusUrl)
  .with(staticToken(directusToken))
  .with(rest());
```

### Creating the Dynamic Route

Let's create the route that will be used to redirect the user to the slug URL queried from Directus. In the `App.tsx` file add the following code:

```typescript
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { useParams } from "react-router-dom";

function App() {
  return (
    <>
      <h1>Link Shortener</h1>
      <BrowserRouter>
        <Routes>
          <Route path="/:slug" element={<LinkRoute />}></Route>
        </Routes>
      </BrowserRouter>
    </>
  );
}

function LinkRoute() {
  const { slug } = useParams();

  console.log(slug);

  return (
    <>
      <h1>{slug}</h1>
    </>
  );
}

export default App;
```

### Querying Directus

Query the associated record for the slug in Directus and redirect the user to the route link.

Enter the following code in `App.tsx` to define the interface for the data that will be returned from Directus:

```typescript
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { useParams } from "react-router-dom";

interface ShortLink { // [!code ++]
  clicks: number; // [!code ++]
  date_created: string; // [!code ++]
  date_updated?: string; // [!code ++]
  id: number; // [!code ++]
  slug: string; // [!code ++]
  sort?: null; // [!code ++]
  url: string; // [!code ++]
  user_created?: string; // [!code ++]
  user_updated?: null; // [!code ++]
} // [!code ++]

function App() {
  return (
    <>
      <h1>Link Shortener</h1>
      <BrowserRouter>
        <Routes>
          <Route path="/:slug" element={<LinkRoute />}></Route>
        </Routes>
      </BrowserRouter>
    </>
  );
}

function LinkRoute() {....}
```

Query Directus in `App.tsx`:

```typescript
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { useParams } from "react-router-dom";
import { directus } from "./util/directus"; // [!code ++]
import { readItems, updateItem } from "@directus/sdk"; // [!code ++]
import { useEffect, useState } from "react"; // [!code ++]

function App(){...}

function LinkRoute() { // [!code ++]
  const { slug } = useParams();
  const [slugError, setSlugError] = useState(""); // [!code ++]

  useEffect(() => { // [!code ++]
    async function fetchShortLink() { // [!code ++]
      try { // [!code ++]
        const data = await directus.request(readItems("short_link")); // [!code ++]
// [!code ++]
        const slug_data = data // [!code ++]
          .map((y) => y) // [!code ++]
          .filter((z) => z.slug.toLowerCase().includes(slug)); // [!code ++]
// [!code ++]
        if (!slug_data || slug_data?.length === 0) { // [!code ++]
          setSlugError(`Invalid Slug: Couldn't find the record →→ ${slug}`); // [!code ++]
          throw new Error("Invalid Slug: Couldn't find that record"); // [!code ++]
        } // [!code ++]
        const shortLink = slug_data[0] as ShortLink; // [!code ++]
// [!code ++]
        await directus.request( // [!code ++]
          updateItem("short_link", shortLink.id, { // [!code ++]
            clicks: shortLink.clicks + 1, // [!code ++]
          }) // [!code ++]
        ); // [!code ++]
// [!code ++]
        window.location.assign(`${shortLink.url}`);// [!code ++]
// [!code ++]
      } catch (error) { // [!code ++]
        console.log(error); // [!code ++]
      } // [!code ++]
    } // [!code ++]
// [!code ++]
    fetchShortLink(); // [!code ++]
  }, [slug]); // [!code ++]

  return (
    <>
      <h1>{slug}</h1> // [!code --]
      {slugError && <h1 style={{ color: "red" }}>{slugError}</h1>} // [!code ++]
    </>
  );
}

export default App;
```

The `LinkRoute` component uses the `useParams` hook to get the `slug` parameter from the URL and the `useState` hook to store an error message if the slug is invalid.

The `useEffect` hook is used to fetch data from the Directus API via the `fetchShortLink` effect function, which performs the following actions:

1. It makes a request to the Directus API to read all short links via the `readItems` composable.
2. The response data is filtered to find the short link that matches the current slug.
3. If no matching short link is found, it sets an error message and throws an error.
4. If a matching short link is found, it updates the short links `click` property by `1` via the `updateItem` composable from Directus API.
5. Finally, the user is redirected to the URL associated with the short link using the `window.location.assign` method.

Finally, if an error occurs during the data fetching process, the error is caught and logged to the console. The `LinkRoute` component returns the error message if the slug is invalid.

## Conclusion

You’ve just learned how to set up the data collection process and implement dynamic routing using React and Directus. The broad process involved setting up a Directus data model, customizing user roles and permissions, creating an API endpoint to query data from Directus, and configuring a dynamic route to navigate to slug URLs.

I hope you find this tutorial useful - if you have any questions or hurdles feel free to join the Directus [official Discord server](https://directus.chat).
