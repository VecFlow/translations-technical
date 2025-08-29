# Translation Workflow Blurb

As part of our technical interview process, we share some coding challenge details in advance. At the start of the interview, you’ll receive:

- A **backend specification** describing how the Translation API works
- A **Figma mockup** of the frontend flow

### The Task

You’ll build a Next.js React frontend that implements the translation workflow.

- The backend route will be public
- It will return an ID that you’ll use to fetch updates from Supabase
- We’ll provide Supabase keys so you can implement Supabase Realtime

You are encouraged to use AI-powered tools (e.g. Cursor, ChatGPT), documentation, and the internet as you would in day-to-day work. We’re interested in how you reason through the problem and translate a loose specification into working code. It’s completely fine if you’re not already familiar with Next.js or Supabase—we expect you’ll work through documentation as needed.

### What to Build

The backend spec provides a Translation API that supports both document and text translation, with:

- Streaming updates for text translations
- Polling and realtime updates through Supabase

Your first goal is to implement the data-fetching layer for only text:

- Create translation requests
- Subscribe to live updates
- Display translation progress for text

Think of a flow similar to Google Translate, where users paste text, choose a target language, and see results appear in real time.

### Next Steps

Once data fetching works, shift focus to UI and UX polish:

- Stream translations progressively into the UI
- Handle edge cases gracefully (errors, timeouts, unsupported formats)
- Make the experience smooth and intuitive

This exercise is meant to be realistic. When we built our own translation system, we also had to learn new streaming frameworks and work through unfamiliar documentation. We’re not judging for pixel-perfect design—but rather how you take a backend spec and a lightweight product idea and turn it into a usable, working frontend workflow.
