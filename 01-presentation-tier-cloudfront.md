# Presentation Tier — Website Delivery with CloudFront

**Stage 1 of 3** · Services: Amazon S3, Amazon CloudFront, Origin Access Control
**Build time:** ~2 hours

---

## Goal

Serve a static web front end globally with low latency, without exposing the underlying storage to the public internet.

This is the presentation tier of the three-tier application — the layer the user actually touches. It calls the logic tier (documented in stage 2) to fetch data.

---

## Architecture

```
   User (anywhere)
         │
         │  HTTPS
         ▼
   CloudFront distribution
   ├─ Default root object: index.html
   ├─ Edge caching at global locations
   └─ Origin Access Control (OAC)
         │
         │  signed request — the only permitted path
         ▼
   S3 bucket  (Block Public Access ON)
   ├─ index.html   structure
   ├─ style.css    presentation
   └─ script.js    form submission / interactivity
```

---

## What was built

**1. S3 as the origin, not the host.**
The bucket stores three files: `index.html` for structure, `style.css` for styling, and `script.js` for interactive elements such as form submission. CloudFront is a content delivery network, not a storage service — it distributes objects that already live somewhere, so S3 does the storing and CloudFront does the delivering.

Before uploading, the site was validated by opening `index.html` locally in a browser to confirm it rendered without errors.

**2. The CloudFront distribution.**
A distribution is the set of instructions telling CloudFront how to deliver the content. The S3 bucket was set as the origin, and the default root object was set to `index.html` so that a request to the root URL returns the page rather than a listing or an error.

**3. Origin Access Control.**
OAC is a dedicated identity that lets CloudFront read from S3 while the bucket itself stays closed to the public. This is what keeps the objects private while still serving them worldwide.

---

## What broke

### `Access Denied` on the distributed site

Visiting the distribution returned an access-denied error. The distribution's origin access setting was **public**, which assumes the origin bucket is also public — and it was not.

The fix was to switch origin access to Origin Access Control. But that alone was not sufficient: **creating the OAC does not by itself grant CloudFront access.** The objects remained private to everyone, CloudFront included. Creating the OAC generates a bucket policy, and that policy has to be applied to the bucket before the distribution can read the objects.

That two-step nature — create the identity, then grant it — is the part worth remembering. It's the same pattern that appears again in the data tier with IAM.

### `403 Forbidden` on S3 static website hosting

While comparing approaches, S3 static website hosting was enabled without making the objects publicly available, producing a 403. Unchecking "Block public access" alone did not resolve it; the bucket policy also had to explicitly allow public read access. The site only became reachable after both the block setting and the policy were correct, with the right bucket name in the policy.

---

## Project extension: S3 hosting vs. CloudFront

This comparison was done beyond the guided steps, and it produced the clearest finding of the build.

**Security.** Serving the site directly from S3 static website hosting requires making the bucket's objects publicly readable. Serving it through CloudFront with OAC means bucket access is restricted to the distribution alone. The CloudFront path is the more secure of the two, and it isn't close.

**Performance.** Load time is how long the browser takes to receive and display content. Measured against each other, the CloudFront-delivered site was faster than the S3-hosted one, because content is cached at edge locations near the user rather than fetched from the single bucket region every time.

**When each fits.** A business that cares about performance — streaming, media-heavy sites, anything with a geographically spread audience — wants CloudFront. S3 static website hosting is sufficient for a simple site with no load-time requirements and no sensitivity about public objects.

---

## What I'd add before this carried real traffic

- A custom domain with an ACM certificate, rather than the default distribution domain
- A cache policy tuned to the content, instead of defaults
- Cache invalidation as part of a deployment process, so updates propagate predictably
- Security headers via a response headers policy
- Access logging on the distribution
