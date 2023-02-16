The archive is split into 4 parts that are less than 100MB each to stay under
the Github LFS size limit.

To restore the full archive from all 4 parts, run the following (on MacOS):
```sh
zip -s0 twitter-slack-emojis-4splits.zip --out twitter-slack-emojis.zip
```
This will put `twitter-slack-emojis.zip` in the current folder which can
then be opened with `unzip` or Archive Utilities.
