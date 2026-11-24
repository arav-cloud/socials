# Blog-to-Instagram-and-X Publisher

An n8n workflow that accepts a blog URL, extracts the page's metadata and text, generates social copy with an OpenAI model, emails the drafts for approval, and publishes an approved image post to Instagram.

> **Current workflow status:** The workflow generates an X post and includes it in the approval email, but it does **not** publish to X. There are no X/Twitter nodes or credentials in the provided workflow. Instagram publishing is the only configured publishing path.

## Workflow at a glance

```text
Submit Blog URL
  → Fetch Blog Page
  → Extract Blog Content
  → Write Social Posts
  → Prepare Posts
  → Ask for Approval (email)
  → Approved?
      └─ approved → Create Instagram Media → Publish Instagram Post
```

If approval is declined, the workflow has no connected false branch, so it ends without publishing.

## Complete process

1. **Submit Blog URL — Form Trigger**
   - Displays a form titled “Share a blog post on Instagram and X”.
   - Requires a `Blog URL` field (`blog_url`).
   - The configured form path is `blog-to-social-publisher`.

2. **Fetch Blog Page — HTTP Request**
   - Requests the URL submitted in `blog_url`.
   - Configures the response as text so the page's HTML can be passed to the extraction step.

3. **Extract Blog Content — HTML Extract**
   - Reads the page HTML and extracts:
     - `title` from the `<title>` element.
     - `image_url` from the `content` attribute of `meta[property="og:image"]`.
     - `description` from the `content` attribute of `meta[name="description"]`.
     - `body_text` from `article, main, body`.
   - The site needs to expose suitable metadata and an accessible cover image for the best results. If the selectors do not match or metadata is absent, values may be empty.

4. **Write Social Posts — OpenAI**
   - Sends the extracted title, description, and up to the first 6,000 characters of body text to the configured OpenAI model (`gpt-6.1-terra`).
   - Requests a JSON object with exactly two fields:
     - `tweet`: an X post of at most 230 characters, with no URL and at most two hashtags.
     - `instagram_caption`: an engaging caption in 3–5 short lines; emojis are allowed.
   - The model response is expected in the node's `output[0].content[0].text` field.

5. **Prepare Posts — Set**
   - Parses the model's JSON response (also stripping optional JSON Markdown fences).
   - Creates `tweet_text` by appending the submitted blog URL to the generated tweet.
   - Carries forward `instagram_caption`, the extracted `image_url`, and `title`.
   - Although the prompt says the tweet itself should contain no URL, the prepared `tweet_text` deliberately appends the blog URL for the approval draft.

6. **Ask for Approval — Gmail Send and Wait**
   - Emails the X draft, Instagram caption, and image URL for review.
   - The recipient is currently hard-coded as `approver@example.com`; replace this with the intended reviewer before using the workflow.
   - The Gmail node pauses for a response and passes the approval result to the next step.

7. **Approved? — If**
   - Checks whether `data.approved` is boolean `true`.
   - Only the true output is connected. A declined or otherwise non-approved response stops the workflow without publishing.

8. **Create Instagram Media — Facebook Graph API**
   - Sends an Instagram Graph API `POST` to the configured Instagram account's `media` edge.
   - Supplies the extracted `image_url` and generated `instagram_caption` to create a media container.
   - The configured Instagram account ID is set in the node. Use an account and credentials that are authorized for Instagram Graph API publishing.

9. **Publish Instagram Post — Facebook Graph API**
   - Sends a `POST` to the same account's `media_publish` edge.
   - Uses the `id` returned by the media-creation request as `creation_id` to publish the container.

## Setup requirements

Before activating the workflow, configure credentials and replace the example values:

- **OpenAI:** Set up the OpenAI credential used by the “Write Social Posts” node and confirm that the selected model is available to your account.
- **Gmail:** Set up the Gmail OAuth2 credential used by “Ask for Approval”; set the reviewer email in that node.
- **Meta / Instagram:** Set up a Facebook Graph API credential with the required permissions and configure the intended Instagram professional account ID. The account must meet Meta's publishing requirements.
- **Blog URL:** Test with pages that are publicly reachable from the n8n instance and provide valid Open Graph image metadata.
- **Workflow activation:** Import the workflow JSON into n8n, configure credentials, test the full approval flow, and then activate the form trigger.

Credential names and IDs embedded in an exported workflow are instance-specific references, not credentials that another n8n instance can use directly. Re-select or create credentials after importing. Do not commit access tokens, OAuth secrets, or other credentials to the repository.

## Adding X publishing

The current workflow only drafts X copy; it does not post it. To publish to X, add an X-compatible publishing step after the **Approved?** node's true output (or after Instagram publishing, depending on the desired behavior). Configure the X API credentials and map the prepared `tweet_text` into the post's text field. Consider whether Instagram and X should publish independently so a failure on one platform does not prevent the other.

## Operational and safety notes

- The form accepts a URL that is fetched by the n8n server. In a production deployment, validate and restrict allowed schemes and hosts to trusted public sites, and block access to private, loopback, link-local, and internal network addresses. This helps prevent server-side request forgery (SSRF).
- Validate that the extracted image is publicly fetchable and satisfies Instagram's current media requirements before publishing.
- Treat blog content and model output as untrusted input. Review generated copy and image details before approval; the workflow's email approval step is an intentional publishing gate.
- Consider adding error handling for inaccessible pages, missing metadata, invalid model JSON, email failures, Instagram API errors, and publish failures.
- The workflow sends the submitted blog URL with the X draft, but the generated Instagram caption does not automatically include the URL.

## Node inventory

| Node | Purpose |
| --- | --- |
| Submit Blog URL | Collects a required blog URL through an n8n form. |
| Fetch Blog Page | Fetches the submitted page as text/HTML. |
| Extract Blog Content | Extracts title, Open Graph image, description, and page text. |
| Write Social Posts | Generates the X draft and Instagram caption. |
| Prepare Posts | Parses the generated JSON and assembles fields for review and publishing. |
| Ask for Approval | Emails the drafts and waits for an approval response. |
| Approved? | Allows publishing only when `data.approved` is `true`. |
| Create Instagram Media | Creates an Instagram media container from the image and caption. |
| Publish Instagram Post | Publishes the created Instagram media container. |
