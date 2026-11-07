# MegaBlog

MegaBlog is a full-stack blogging app built with **React** and **Appwrite**. Users can sign up, log in, write posts in a rich-text editor, attach a featured image, and edit or delete their own posts. Appwrite handles authentication, the database and file storage, so the project has no custom backend server.

## Features

- **Authentication**: sign up, log in and log out with email and password (Appwrite Auth). The session survives page reloads.
- **Protected routes**: only logged-in users can see All Posts, Add Post and Edit Post; logged-in users are sent away from Login and Signup.
- **Create posts**: a title, an auto-generated URL slug, rich-text content (TinyMCE), a featured image and an active/inactive status.
- **Edit and delete**: only the author of a post sees the Edit and Delete buttons. Replacing or deleting a post also deletes its old image from storage.
- **Home feed**: shows every post whose status is `active`.
- **Global auth state**: Redux Toolkit stores whether the user is logged in and who they are.
- **Styling**: Tailwind CSS utility classes.

## Tech stack

| Area | Tool |
|---|---|
| UI | React 19 |
| Build and dev server | Vite |
| Routing | React Router (`createBrowserRouter`) |
| State | Redux Toolkit + React Redux |
| Forms | React Hook Form |
| Rich-text editor | TinyMCE (`@tinymce/tinymce-react`) |
| Rendering post HTML | html-react-parser |
| Backend (auth, database, storage) | Appwrite (Web SDK) |
| Styling | Tailwind CSS 3 with PostCSS and Autoprefixer |
| Linting | ESLint |

## Project structure

```
MegaBlog/
├── index.html               # HTML shell that loads src/main.jsx
├── .env                     # Appwrite and TinyMCE keys (not committed)
├── tailwind.config.js       # Which files Tailwind scans for classes
├── postcss.config.js        # Runs Tailwind and Autoprefixer through Vite
├── vite.config.js
└── src/
    ├── main.jsx             # Entry point: Redux store, router and all routes
    ├── App.jsx              # Root layout: checks the session, renders Header / page / Footer
    ├── App.css              # Tailwind directives
    ├── conf/
    │   └── conf.js          # Reads the VITE_* environment variables in one place
    ├── appwrite/
    │   ├── auth.js          # AuthService: create account, login, logout, current user
    │   └── config.js        # Service: post CRUD (database) and image upload (storage)
    ├── store/
    │   ├── store.js         # Redux store
    │   └── authSlice.js     # auth state: { status, userData } with login / logout actions
    ├── components/
    │   ├── index.js         # Re-exports every component for short imports
    │   ├── Header/          # Header.jsx (navigation) and LogoutBtn.jsx
    │   ├── Footer/          # Footer.jsx
    │   ├── container/       # Container.jsx: centred, max-width wrapper
    │   ├── post-form/       # PostForm.jsx: create and edit form
    │   ├── AuthLayout.jsx   # Route guard (exported as AuthLayout and Protected)
    │   ├── Login.jsx        # Login form
    │   ├── Signup.jsx       # Signup form
    │   ├── RTE.jsx          # TinyMCE editor wired into React Hook Form
    │   ├── PostCard.jsx     # Post preview card (image + title)
    │   ├── Input.jsx        # Reusable labelled input
    │   ├── Select.jsx       # Reusable labelled select
    │   ├── Button.jsx       # Reusable button
    │   └── Logo.jsx
    └── pages/
        ├── Home.jsx         # Active posts feed
        ├── AllPosts.jsx     # Every post
        ├── AddPost.jsx      # New post (PostForm)
        ├── EditPost.jsx     # Edit post (PostForm filled with the post)
        ├── Post.jsx         # Single post view
        ├── Login.jsx        # Wraps the Login component
        └── Signup.jsx       # Wraps the Signup component
```

## How it works

### Routes

All routes render inside `App`, which shows the Header and Footer around the current page (`<Outlet />`).

| Path | Page | Who can open it |
|---|---|---|
| `/` | Home | Everyone |
| `/login` | Login | Logged-out users only |
| `/signup` | Signup | Logged-out users only |
| `/all-posts` | All Posts | Logged-in users only |
| `/add-post` | Add Post | Logged-in users only |
| `/edit-post/:slug` | Edit Post | Logged-in users only |
| `/post/:slug` | Single post | Everyone |

`AuthLayout` guards the routes. With `authentication={true}` it sends logged-out users to `/login`; with `authentication={false}` it sends logged-in users to `/`.

### Authentication flow

1. **On page load**, `App.jsx` calls `authService.getCurrentUser()`. If Appwrite has a session, it dispatches `login({ userData })`; otherwise it dispatches `logout()`. The app shows nothing until this check finishes, so routes never flash the wrong state.
2. **Signup** calls `account.create(...)` and then logs the user in straight away.
3. **Login** calls `account.createEmailPasswordSession(...)`, fetches the user, and stores it in Redux.
4. **Logout** calls `account.deleteSessions()` and dispatches `logout()`.

When nobody is logged in, the browser console shows a 401 error, `User (role: guests) missing scopes (["account"])`, from the page-load check. This is expected: a guest has no account to fetch.

### Redux state

```js
state.auth = {
  status: false,     // true when logged in
  userData: null,    // the Appwrite user object ($id, name, email, ...)
}
```

`Header` reads `status` to choose which links to show, and `Post` compares `userData.$id` with the post's `userId` to decide whether to show Edit and Delete.

### Posts

Each post is a document in an Appwrite collection. **The slug is the document ID**, so `/post/my-first-post` loads the document with ID `my-first-post`.

`PostForm` is used both to create and to edit:

- Typing a title fills the slug automatically: lower-case, with spaces and symbols replaced by `-`.
- **Create**: uploads the image to the storage bucket, then creates the document with the image's file ID as `featuredImage` and the current user's ID as `userId`.
- **Edit**: if a new image is chosen, uploads it and deletes the old one, then updates the document. The slug can't be changed after creation.

`Post.jsx` renders the saved HTML content with `html-react-parser`, and images are shown with `getFileView(fileId)`.

### Appwrite service methods

| Method | File | What it does |
|---|---|---|
| `createAccount({ email, password, name })` | `auth.js` | Creates a user, then logs in |
| `login({ email, password })` | `auth.js` | Creates an email/password session |
| `getCurrentUser()` | `auth.js` | Returns the logged-in user, or `null` |
| `logout()` | `auth.js` | Deletes all of the user's sessions |
| `createPost({...})` | `config.js` | Creates a post document (ID = slug) |
| `updatePost(slug, {...})` | `config.js` | Updates title, content, image and status |
| `deletePost(slug)` | `config.js` | Deletes a post; returns `true` or `false` |
| `getPost(slug)` | `config.js` | Returns one post, or `false` |
| `getPosts(queries)` | `config.js` | Lists posts; defaults to `status = "active"` |
| `uploadFile(file)` | `config.js` | Uploads an image to the bucket |
| `deleteFile(fileId)` | `config.js` | Deletes an image from the bucket |
| `getFileView(fileId)` | `config.js` | Returns the image URL |

## Getting started

### Prerequisites

- Node.js 20 or newer
- A free [Appwrite Cloud](https://cloud.appwrite.io) account (or a self-hosted Appwrite)
- A free [TinyMCE](https://www.tiny.cloud) API key

### 1. Install

```bash
git clone https://github.com/DSheth2004/MegaBlog.git
cd MegaBlog
npm install
```

### 2. Set up Appwrite

In the Appwrite console:

1. **Create a project.** Copy its **Project ID** and **API endpoint** (for example `https://cloud.appwrite.io/v1`).
2. **Add a Web platform** with the hostname `localhost`, so the browser is allowed to call Appwrite.
3. **Auth**: make sure the **Email/Password** method is enabled.
4. **Create a database**, then a **collection** inside it, with these attributes:

   | Attribute | Type | Required | Notes |
   |---|---|---|---|
   | `title` | String | Yes | e.g. size 255 |
   | `content` | String | Yes | Large size (e.g. 10000+), since it stores HTML |
   | `featuredImage` | String | Yes | The storage file ID, size 255 |
   | `status` | String | Yes | `active` or `inactive` |
   | `userId` | String | Yes | The author's user ID |

5. In the collection's **Indexes** tab, add a **key index on `status`**. Home filters on it, and the query fails without the index.
6. In the collection's **Settings → Permissions**, add the role **Users** with **Create, Read, Update and Delete**. Without this, saving posts returns 401.
7. **Create a storage bucket.** In its **Settings → Permissions**, give **Users** the same four permissions, and allow image file types (`png`, `jpg`, `jpeg`, `gif`).

> Viewing posts while logged out needs **Read** for the **Any** role on the collection and the bucket. Without it, Home shows "Login to read posts" to guests.

### 3. Environment variables

Create a `.env` file in the project root:

```env
VITE_APPWRITE_URL="https://cloud.appwrite.io/v1"
VITE_APPWRITE_PROJECT_ID="your-project-id"
VITE_APPWRITE_DATABASE_ID="your-database-id"
VITE_APPWRITE_COLLECTION_ID="your-collection-id"
VITE_APPWRITE_BUCKET_ID="your-bucket-id"
VITE_TINYMCE_API_KEY="your-tinymce-api-key"
```

Vite only exposes variables that start with `VITE_`. `.env` is listed in `.gitignore`, so your keys stay out of Git. Restart the dev server after changing it.

### 4. Run

```bash
npm run dev
```

Open http://localhost:5173.

## Scripts

| Command | What it does |
|---|---|
| `npm run dev` | Starts the dev server with hot reload |
| `npm run build` | Builds the production files into `dist/` |
| `npm run preview` | Serves the built `dist/` folder locally |
| `npm run lint` | Runs ESLint |

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| 401 `missing scopes (["account"])` in the console when logged out | Expected; see [Authentication flow](#authentication-flow). |
| 401 when creating a post or uploading an image | The **Users** role is missing permissions on the collection or the bucket (setup steps 6 and 7). |
| Home is empty even though posts exist | Add the `status` index (step 5), and check that the posts are `active`. |
| CORS error calling Appwrite | Add a Web platform for `localhost` (step 2). |
| Tailwind classes have no effect | Make sure `postcss.config.js` exists, then restart `npm run dev`. |
| The editor shows an API-key warning or is read-only | Set `VITE_TINYMCE_API_KEY` and add `localhost` to the approved domains in your TinyMCE account. |
| Images don't show | Check the bucket's Read permission. Appwrite's free plan doesn't include image transformations, which is why the app uses `getFileView` rather than `getFilePreview`. |
| Build works on Windows but fails on Linux | Import paths are case-sensitive on Linux; the folder is `container`, not `Container`. |
