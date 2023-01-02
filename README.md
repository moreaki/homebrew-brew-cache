[Homebrew External Command](https://docs.brew.sh/External-Commands)

Search for locally installed package(s) that own the file pattern, similar to

* dpkg --search PATTERN
* apt-cache search PATTERN
* yum whatprovides PATTERN
* pacman --query --owns PATTERN

## Install

```
% brew tap moreaki/brew-cache
```

It works on macOS as well as Linux (untested in this version). It's minimally invasive 
(essentially only one file is created and maintained with a linear lookup list).

## Usage

### See help

```
% brew cache
Usage: brew cache [options]

Query the local Homebrew packages cache

   -u            Update/build the local packages cache
   -s  pattern   Show package(s) owning pattern
                                   files
   -d            Display debugging information
   -i            Display cache information
   -q            Quiesce output
   -r            Remove actual cache
   -h, --help    Show this message
```

### Additional calling conventions

You can also use the following temporary environment variables to control the output flow of the tool:
```
% QUIET=1 brew cache -i                     # instead of `brew cache -q -i` 
% DEBUG=1 brew cache -u                     # instead of `brew cache -d -u`
```

Additional environment variables:
```
% CACHE_DIR=/tmp brew cache -u              # Set another base directory for the cache
% LIMIT_TO_PKGS="scc|srt" brew cache -u     # Build cache only with the packages in the list
```

### CRUD operations on the packages cache

```
% brew cache -i
[d56813b] Cache dir : /Users/moreaki/Library/Caches/com.github.ten0s.brew-cache
[d56813b] Cache hash: d56813baa5270d5e333708cabf85fa17c900bc0b
[d56813b] Cache file: /Users/moreaki/Library/Caches/com.github.ten0s.brew-cache/homebrew-d56813baa5270d5e333708cabf85fa17c900bc0b.cache
[d56813b] Cache size: 135702 file lookup entries
[d56813b] Cache pkgs: 193 packages managed
```

```
% brew cache -u
[d56813b] Cache is up-to-date
```

Let's remove/delete the current cache:
```
% brew cache -r
[d56813b] Removing cache with hash d56813baa5270d5e333708cabf85fa17c900bc0b
```

After removal, the update works as follows:
```
% brew cache -u
[] No cache is available
[] Removing cache with hash
[d56813b] Building cache
Warning: Treating handbrake as a formula. For the cask, use homebrew/cask/handbrake
Warning: Treating jasper as a formula. For the cask, use homebrew/cask/jasper
Warning: Treating snappy as a formula. For the cask, use homebrew/cask/snappy
[d56813b] New cache is built
```

If the git HEAD hash of homebrew updates, there might be a need to update the cache as well:
```
% brew update
Updated 3 taps (homebrew/core, homebrew/cask and homebrew/services).
==> New Casks
calhash                                            catoclient
==> Outdated Formulae
libmagic
[...]
% brew cache -u
[d56813b] Updated git hash (e12507b) is available
[d56813b] Removing cache with hash d56813baa5270d5e333708cabf85fa17c900bc0b
[e12507b] Building cache
Warning: Treating handbrake as a formula. For the cask, use homebrew/cask/handbrake
Warning: Treating jasper as a formula. For the cask, use homebrew/cask/jasper
Warning: Treating snappy as a formula. For the cask, use homebrew/cask/snappy
[e12507b] New cache is built
```

### Search operations

The search is for a locally installed package that owns the given file. The pattern can 
be a simple string or a regular expression.

### Caveats

- Currently, the cache building process takes a long time because brew itself is not 
  very fast, and because no parallel cache building has been introduced.
- The current cache is unfortunately incomplete on M1 and M2 Apple machines, 
  since homebrew has no (easily available) intrinsic notion of maintaining the file 
  list of installed packages. Especially the symlinks of binaries are not in the cache.
- The lookup is a simple sed on a text file, with all it's limitations.
- No partial update is possible after installing a package. The cache always needs to 
  be rebuilt. Ideas are to use brew hooks in bash or inotify or implement the whole thing
  using a BerkleyDB.
- Even if nothing has changed besides an update of the homebrew git HEAD hash reference
  the tool will rebuild the whole cache. Since it's still very slow for systems with many
  packages, this can lead to a lower user acceptance.

## Implementation Details and Design

The cache essentially is a list of filename with their corresponding package name and 
version as follows:
```
% head -10 /Users/moreaki/Library/Caches/com.github.ten0s.brew-cache/homebrew-e12507bc06a3b3d3055a520e9f4461bff890bda4.cache/opt/homebrew/Cellar/ack/3.6.0/INSTALL_RECEIPT.json:ack:3.6.0
/opt/homebrew/Cellar/ack/3.6.0/bin/ack:ack:3.6.0
/opt/homebrew/Cellar/ack/3.6.0/.brew/ack.rb:ack:3.6.0
/opt/homebrew/Cellar/ack/3.6.0/share/man/man1/ack.1:ack:3.6.0
/opt/homebrew/Cellar/aom/3.5.0_1/INSTALL_RECEIPT.json:aom:3.5.0_1
/opt/homebrew/Cellar/aom/3.5.0_1/LICENSE:aom:3.5.0_1
/opt/homebrew/Cellar/aom/3.5.0_1/bin/aomdec:aom:3.5.0_1
/opt/homebrew/Cellar/aom/3.5.0_1/bin/aomenc:aom:3.5.0_1
/opt/homebrew/Cellar/aom/3.5.0_1/.brew/aom.rb:aom:3.5.0_1
/opt/homebrew/Cellar/aom/3.5.0_1/CHANGELOG:aom:3.5.0_1
```

The search of a pattern `SEARCH` in the `$CACHE_FILE` is a simple `sed` as follows:
```
sed -rn -- "s@(.*)${SEARCH}(.*):(.*):(.*)@\3 \4@p" "$CACHE_FILE" | sort -u
```

The design considerations are:

- Minimally invasive
- Simple usage as close to the Linux package manager's search functionalities

## License

The project is licensed under the 2-Clause BSD License.<br>
See [LICENSE](LICENSE) or
https://spdx.org/licenses/BSD-2-Clause.html
for full license information.
