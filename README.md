# articles

Public source archive for the articles published on HaoyangMa.

Articles use Hugo page bundles and are grouped by first publication date:

```text
articles/
|-- _index.md
`-- YYYY/
    `-- MM/
        `-- slug/
            |-- index.md
            |-- images/    (optional)
            `-- files/     (optional)
```

Publishing workflow:

1. Write or update an article in this repository.
2. Preview or build the Hugo site from the hymspace repository.
3. Commit and push this repository first.
4. Update the hymspace submodule pointer and push the website repository.
5. Deploy the website after both repositories are synchronized.

Articles are published under All Rights Reserved unless an article states
otherwise.
