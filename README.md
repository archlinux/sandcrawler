# Archweb out of date handling

Automate the out of date flagging functionality in archweb for all the packages
in our repository with an external service which crawls the web to find new
versions of packages.


## Requirements

- easily add a new package to the watch list
- a way to run this checker locally for debugging
- control over actual watch targets to really track exactly what we package (f.e. fork etc)
- have a page/overview of all available versions (including non-latest and RC/alpha/beta)
- filter out RC/alpha/beta by default but allow to override (f.e. dmraid)
- automatically flag the package OOD if the latest filtered version is greater (take epoch into account)
- Have a possibility to specify multiple watches per pkgbase, like both: github + pypi
- add transformation rules to convert upstream fetched version string to a variant used in pkgver
- detect unstable releases like gnome even/uneven


## Architecture

The overall tooling and daemon should not be part of archweb, but instead a
specific format shall be defined that interfaces the tool and archweb itself.
This allows decoupling and reduce complexity for handling this in archweb and
makes it easier runnable and testable locally and on a dedicated machine.


### Daemon

- recursively walk through the repository checkout crawling watch files for pkgbase
- read the watch file and translate it into actual actions
- crawl the target endpoint and acquire all versions
- transform the results into the interface format
- call archweb API endpoint using a token to push the current results for a package
- 20 goto 10


### Archweb

- provides API endpoint to retrieve package version information
- reads the input and puts it into the database
- trigger check after push to test greatest version against current and flag package OOD
- overview page per pkgbase to list a table of all provides and all versions + mark greatest
- add a archweb developer report page that lists packages which are missing watch files


#### Database model

A new table **providers** is created with the upstream providers of versions:

| id          |     name     |
| ----------  | ------------ |
| primary key | charfield    |

Example table

| id          |     name     |
| ----------- | ------------ |
| 1           | "url"        |
| 2           | "github"     |
| 3           | "pypi"       |

The default to "url" as provider.

A new table **packageversions** is created which contains the found version per pkgbase:

| pkgbaseid   | providerid   | version     |
| ----------- | ------------ | ----------- |
| foreign key | foreign key  | charfield   |

| pkgbaseid   | providerid   | version     |
| ----------- | ------------ | ----------- |
| 1           | 1            | 0.2         |
| 1           | 2            | 0.2         |


### Format

A list of JSON entries consisting the pkgbase and providers with the versions which are known.

```json
[{
    "pkgbase": "python-pcapy",
    "providers": {
        "github": ["1.0", "1.1", "2.0-alpha"],
        "pypi": ["1.0", "1.1", "2.1"],
    }
}]
```


## Considerations

### Repology

#### Advantage
- Already tracks lots of projects

#### Disadvantage
- Only tracks versions of distro releases, no real upstream checking

#### Result
- Dismissed as we want to do real upstream checking

### release-monitoring

#### Advantage
- Already track of lots of project

#### Disadvantage
- the API doesn't give us enough control for more complex use cases
- there are lot of loop holes that we found while testing, like hexedit and bunch of others
- too little and indirect control of watching the actual thing we package

### Result
- Dismissed as we seek to have better control over checking exactly the upstream we are actually packaging

### urlwatch

#### Advantage
- Write your own custom site checkers

#### Disadvantage
- Need to write custom site checkers for Github and other sources
- Requires a config file with urls and a database state

#### Result

### nvchecker

#### Advantages
- Regex flexibility with "to/from" patterns and "include/exclude" patterns
- Provides quite a lot of check providers like pypi, github, cpan

#### Disadvantages
- Manually add upstreams and configs

### Debian's watch file

Debian's watch file mechanism is used to fetch the latest version of software
it's not for watching for new releases.

https://wiki.debian.org/debian/watch
https://salsa.debian.org/debian/devscripts/blob/master/scripts/uscan.pl

### Advantages

### Disadvantages
- The tool is intended to get the latest tarball of a based on the content of a Debian watch file. Not for reporting a newer version
