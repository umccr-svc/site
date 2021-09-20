# UMCCR website

IMPORTANT: Clone this repo with `--recursive`, otherwise the themes will not be pulled and the builds will fail.

```bash
$ git clone --recursive https://github.com/umccr-svc/site
```
## Add Author
  
## Where to place my content

Drop your `.md` files under `content/` and the images under `static/img/<YEAR>/<MONTH>`. Use common sense and use the already published content as a template.

In order to view your changes locally (before pushing it live), just use:

```bash
$ ./view.sh
```

Unless you are using RStudio+blogdown Addin already. Then just git push and the changes should be visible at:

https://umccr.org/

If you cannot see your changes, please check that there are no deployment errors either locally or at netlify.com Deploy interface.
