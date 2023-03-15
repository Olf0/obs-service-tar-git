# obs-service-tar-git
[**Jolla's OBS `tar_git` source service**](https://github.com/MeeGoIntegration/obs-service-tar-git)

Note that this README is not an exhaustive description or even a specification.  It is merely a user contributed write-up providing some guidance how `tar_git` works.  This write-up may have flaws or be fully erroneous at places; if you want to know something for sure, please read `tar_git`'s [interface definition](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/master/tar_git.service) and its [shell script](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/master/tar_git), which provides all its functionality.

Side note: This is *not* about the regular OBS (Open Build Service) services (plug-in modules) `tar_scm` or `git_tar`.  The only publicly visible place where `tar_git` runs is the SailfishOS-OBS at [build.sailfishos.org](https://build.sailfishos.org/) (= [build.merproject.org](https://build.merproject.org/)), where it is the primary and mandatory service.

## Parameters for OBS `_service` files
See `tar_git`'s [interface definition](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/master/tar_git.service), its [usage output](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/3affb34c2c1d0f565afec8761f451e67cf46661b/tar_git#L69-L83) for descriptions of the parameters and an [exemplary `_service`](https://build.merproject.org/package/view_file/sailfishos:chum:testing/harbour-storeman-installer/_service) file.

## Tag parsing
1. A valid tag for `tar_git` may be prefixed by either a small "v" or a "vendor string" ending in a slash ("/"), see [first part of the function `get_tagver_from_tag`](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/3affb34c2c1d0f565afec8761f451e67cf46661b/tar_git#L376-L387): Both are cut out, a "v" is discarded, but a "vendor string" is prepended (without the slash) to the release field in the spec file (but see 6.).
2. A valid tag for `tar_git` may contain at most a single dash ("-").  If the sub-string starting with the dash conforms to the extended regular expression (ERE) `\-[[:alnum:]]+$`, it is put into the release field (without the dash and directly appended to a "vendor string") and the part up to the dash into the version field by `tar_git`.  Hence a valid tag can be used to fully rewrite the version and release fields in the spec file, before OBS builds this package (but see 6.), see [second part of the function `get_tagver_from_tag`](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/3affb34c2c1d0f565afec8761f451e67cf46661b/tar_git#L389-L396).
3. The resulting, stripped tag must conform to the extended regular expression (ERE) `^[[:digit:]]+\.[[:alnum:].~+]*$` (but is actually [written much more complicated](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/3affb34c2c1d0f565afec8761f451e67cf46661b/tar_git#L406)) and originate from the branch provided by the `branch` parameter, otherwise it is discarded.
4. If there is no valid tag or if a commit ID (short-hash) was provided, or if a `branch` parameter is set and either no `revision` is set or in this Git branch commits exist newer than the tag referenced by the `revision` parameter, `tar_git` [looks for the "closest" tag in the function `set_versha`](https://github.com/MeeGoIntegration/obs-service-tar-git/blob/3affb34c2c1d0f565afec8761f451e67cf46661b/tar_git#L931-L999) which fulfils aforementioned rules (i.e., runs through steps 1 to 3 with it), but misses to filter for the branch provided by the `branch` parameter (i.e., it searches in *all* branches).  Still, it uses the value of the `branch` parameter, or if none is provided, the branch name in which the "closest" tag was found to construct a new version string, which comprises: The processed name of the "closest" tag, a `+`, the branch name, a `.`, a date-time string (`YYYYMMDDhhmmss`), the number of commits since the "closest" tag in the branch it resides enclosed in dots `.<n>.`, a `g` if newer commits exist in the branch referenced by `branch` than the tag referenced by `revision`, and the short-hash of the latest commit, resulting in, e.g., `0.5.2+main.20230129011931.6.g584263a` with `0.5.2-3` as "closest" tag found.  Note that if the "closest" tag determined does not originate from the branch provided by the `branch` parameter, still this parameter is inserted in the constructed string, despite using a Git checkout from another branch.
5. The result of this processing replaces the original `Version:` string in the RPM spec file, which can be viewed on a package's page in the SailfishOS-OBS.
6. Note that, when a package is built by the SailfishOS-OBS, also a new release string is constructed, comprising a `1.`, the package revision at OBS and `.1.jolla`.  This is *not* visible in the processed RPM spec file of a package, because it is done while building a package.

#### Examples
Once understood how `tar_git`'s parsing of Git tags works, one can play with it: For `1.2.3+git1`, `1.2.3~git1` and `1.2.3.git1` the whole string will be used as the processed`<version>` string and become part of the RPM file-name; for `1.2.3-git4`, `git4/1.2.3`  and `git/1.2.3-4` only the `1.2.3` and the release field will be initially set to `git4` for the latter three examples (which is later substituted by `1.I.1`).

As an "real life" example, when using `sfos4.2/0.3.0-5` as a tag name for the `harbour-storeman` app, this results in, e.g., `harbour-storeman-0.3.0-1.13.1.jolla.armv7hl.rpm` as a final package name.

## The `token` parameter
When the `token` parameter contains a non-empty string (not a regular expression), this string is used as a sub-string an original (i.e., unprocessed) Git tag must match to, otherwise the tag is discarded.  Note that such a Git tag is then subjected to the processing described in steps 1 to 4, above.  Hence for the aforementioned example using the `harbour-storeman` app, `sfos` would be a suitable `token`.  Or when marking release versions with the string `release` in Git tags (e.g., `1.2.3-release2`), the string `release` is a suitable `token` to filter for.

## Source archive name
`tar_git` always renames the source archive to `<name>-<version>.<extension>`, using the processed Git tag as `<version>`.  While retaining the name and the extension, there is no way to achieve `<name>-<version>-<release>.<extension>` as its source archive, because the processing prevents any dashes ("-") to be retained in the processed `<version>` string.  If no valid {https|http|ftp} path to a source archive is provided, it performs a Git check-out using the tag determined as described above.

Hence the `%prep` section of a spec file **must** contain [`%autosetup`](https://rpm-software-management.github.io/rpm/manual/autosetup.html) or `%setup [-q] [-n %{name}-%{version}]`, but no other `-n` parameter can be used (which might as well be omitted)!

## Changelog extraction / processing
\<ToDo\>

## Notes
- This is a slightly shortened version of the ["Analysis of tar_git" document in Storeman's wiki](https://github.com/storeman-developers/harbour-storeman/wiki/Analysis-of-tar_git).
