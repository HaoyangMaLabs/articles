HaoyangMa Articles
==================

This repository is the public source archive for the articles published on
HaoyangMa: https://haoyangma.com/

Directory convention
--------------------

Articles use Hugo page bundles and are grouped by their first publication
date:

articles/
|-- _index.md
`-- YYYY/
    `-- MM/
        `-- slug/
            |-- index.md
            |-- images/    (optional)
            `-- files/     (optional)

- YYYY/MM is based on the original publication date.
- slug uses lowercase English words, numbers, and hyphens.
- The article body is always index.md.
- Use draft: true while writing and set draft: false only after review.

Publishing workflow
-------------------

1. Write or update an article in this repository.
2. Preview or build the Hugo site from the hymspace repository.
3. Commit and push this repository first.
4. Update the hymspace submodule pointer and push the website repository.
5. Deploy the website only after both repositories are synchronized.

The website repository includes this repository at content/posts/ as a Git
submodule, so Hugo reads these files directly during the build.

Commit identity
---------------

Commits to this repository use the dedicated author identity:

    HaoyangMa <haoyang@haoyangma.com>

Configure it locally (together with the dedicated SSH key) with:

    git config user.name "HaoyangMa"
    git config user.email "haoyang@haoyangma.com"
    git config core.sshCommand "ssh -i ~/.ssh/id_ed25519_github_second -o IdentitiesOnly=yes"

Copyright notices
-----------------

Do not put copyright notices in article bodies. The website template adds a
uniform copyright block at the end of every article. Use the optional
front matter field `copyright:` only when an article needs different terms.

License
-------

Articles are published under All Rights Reserved unless an article states
otherwise. Public visibility does not grant permission to reproduce,
republish, redistribute, modify, translate, or use the articles commercially.
