# Lab 09: Knowledge check

**Answer each question below it, in your own words, in one to three sentences.** Replace the `...` below **Your answer:** with your text.

## 1. Credentials in the image?

Assume you want to share your image with others and make it publicly available on `ghcr.io`, so that anyone can pull it and run the analysis. A teammate suggests adding `COPY .aws /home/app/.aws` to the `Dockerfile`, so that nobody has to type `-v ~/.aws:/home/app/.aws:ro` any more. (This is a thought experiment: keep your own package private.)

**Your answer:** Give two reasons why this is a bad idea. Think of who could then read your credentials, and of what happens when your Learner Lab session ends.

Copying .aws into the image would be a bad idea for two reasons. First, anyone who can pull the image could read my credentials. Every file a COPY puts into an image stays in its layers, even if a later instruction deletes it, so on a public ghcr.io package anyone could extract ~/.aws/credentials and use my AWS account: read my S3 buckets, and read my SSM parameter with the Atlas password. Second, Learner Lab credentials are temporary. When my session ended during this lab, the old credentials were blocked by the voc-cancel-cred deny policy and the analysis failed with AccessDenied. An image with credentials baked in would stop working after every session and would have to be rebuilt and republished. Mounting ~/.aws at run time with -v ~/.aws:/home/app/.aws:ro keeps the credentials out of the image, and each user supplies their own current credentials.

## 2. Where did the Atlas password come from?

In Step 4, you ran the analysis once with `-v ~/.aws:/home/app/.aws:ro` only, without the three `-e` options. The container still wrote to your Atlas cluster.

*Hint:* Read `get_atlas_settings()` in `src/book_analysis/analyze.py`.

**Your answer:** Explain, step by step, where the container found the Atlas settings in that run.

First, a container does not inherit my terminal’s environment variables. Without the three -e options, MONGODB_ATLAS_URL, MONGODB_ATLAS_USER, and MONGODB_ATLAS_PWD were not set inside the container.Then using, get_atlas_settings() you are able to check these variables first. Since they were missing, it fell back to reading the SSM parameter mongodb_credentials. So, to call SSM, boto3 needed AWS credentials. It found them in /home/app/.aws/credentials, which is my ~/.aws directory mounted read-only into the container. The log line “Found credentials in shared credentials file” shows this. Lastly, the SSM returned a JSON value with url, user, and password, and the script used these to connect to Atlas and save the results.

## 3. Same tag, different image

Run `docker images book-analysis`, and note the image ID of `book-analysis:0.1.0`. Then change the `description` in the `LABEL` of your `Dockerfile`, build again with the same command, `docker build -t book-analysis:0.1.0 .`, and run `docker images book-analysis` again.

**Your answer:** Paste the image ID from before and after the rebuild. Why is it a problem for someone who uses your image that `0.1.0` now names a different image, and what should you have done instead?

Before the rebuild: f6a30940a752
After the rebuild: daf6f2fb390a
All seven build steps came from the cache, so the files in the image are identical. But labels are part of the image’s configuration, so changing the description changed the config and therefore the digest. The tag 0.1.0 stayed the same and now points to a different image. This is a problem for anyone using my image, for someone who pulled 0.1.0 earlier and someone who pulls it now get different images under the same name, so “I ran version 0.1.0” no longer says exactly which image ran. Results are not reproducible, and users can get a changed image without knowing it. Instead, I should treat a released tag as fixed. Bump the version in pyproject.toml and build with the new tag. Anyone who needs an exact image can refer to it by its digest which always names it correctly.


