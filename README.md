# packages
Artix system and world packages

### Notes for maintainers

The packages git follows the archlinux svn structure. It contains both **_core_** and **_extra_** PKGBUILDs, **_system_** and **_world_** in Artix respectively. Packages from **_community_** are accommodated in the `packages-galaxy` git tree.

Each package directory consists of a trunk and repos subdirectories. Artix packages get updated from upstream Arch in trunk (using `buildtree -s`) and any necessary modifications are performed there. Once done, the maintainer can commit/push the updated package into its destination repo and only then the build server will pick up the update and attempt to build the package (i.e. as long as the changes remain in trunk, nothing leaves the room). It's recommended to copy `/etc/artools/artools.conf` to `~/.config/artools/artools.conf` and set the **workspace_dir** option to a place with a few spare GBs of space.

Here's an overview of `buildtree`:
~~~
% buildtree -h
Usage: buildtree [options]
    -p <pkg>      Package name
    -s            Clone or pull repos
    -z            Don't clone or pull arch repos
    -c            Compare packages
    -x            Include unstable kde and gnome
    -u            Show upgrade packages
    -d            Show downgrade packages
    -a            Show testing and staging packages
    -i            Import a package from arch repos
    -t            Import from arch trunk
    -v            View package depends
    -q            Query settings
    -h            This help
~~~


= To sync (clone or pull) the Arch and Artix git repos (use `-sz` to only sync Artix git):

    buildtree -s

= The most interesting option is `-c`. It compares Arch and Artix package versions, combined with `-u` for upgrades and `-d` for downgrades (i.e. shows which packages are newer or older upstream). The testing repos are omitted.

    buildtree -cu
    buildtree -cd

= We can do it in one step:

    buildtree -scu

= To compare Arch and Artix versions in **_gremlins]/[goblins]_** - **_[testing]/[staging_**, use `-a`:

    buildtree -ca

Note, the above check will use the Artix repos as a base, not Arch's. For example, if `fscktheskullofsystemd` is in Artix/**_galaxy_** and Arch/**_community-testing_**, it won't show up in the list.

##### Also, `buildtree` will show the actual repo where a package resides. For example, `wine` belongs to **_multilib_** but in the git tree it's under **_community_**. All `commitpkg` (and its symlinks) operations must be performed using the repo name and not the git subdirectory name.

Now, suppose we saw a shiny package named `foo` in Arch which unfortunately is compiled against _libsystemd.so_ and we want to import it into our repos for proper treatment. We'd like to see some information about it and then import it, so we issue:

    buildtree -v -p foo
    buildtree -p foo -i

and we end up with `foo` in _$workspace_dir/artix/packages/foo/trunk_. Now we can edit the source files and/or the PKGBUILD to our liking. Once we're done, we'll need to commit the changes back to the git tree using `commitpkg`, which standardizes the actions with descriptive commit messages. Keep in mind, `commitpkg` by default updates the trunk; in order to copy the package into an active repo (and inform the build server about it), we must use the appropriate `commitpkg` symlink, as explained below. First, an overview of `commitpkg`:

~~~
$ commitpkg -h
Usage: commitpkg [options]
    -s <name>          Source repository [default:trunk]
    -p <pkg>           Package name
    -r                 Delete from repo (commitpkg only)
    -u                 Push
    -q                 Query settings and pretend
    -h                 This help
~~~

###### commitpkg has following symlinks in place:

- extrapkg 
- corepkg 
- testingpkg 
- stagingpkg 
- communitypkg 
- community-testingpkg 
- community-stagingpkg 
- multilibpkg 
- multilib-testingpkg 
- multilib-stagingpkg

The symlinks above call `commitpkg` which copies the contents of _packages/foo/trunk_ into _packages/foo/repos/$destination_repo/_. Note the use of Arch repo names in the symlinks; this is intentional because it makes it easier to maintain `artools`.

#### Some examples

= ###### After we've imported `foo` from Arch, we want to put it in **_testing_** (i.e. **_gremlins_**). So, we release it from trunk into repos/testing and push with `-u` (in this case `-s trunk` can be omitted, as `-s` defaults to _trunk_):

    testingpkg -p foo -s trunk -u

= ###### Once `foo` has been tested to kingdom come, we decide to move it from **_testing_** to **_core_**:

    corepkg -p foo -s testing -u

The build server will move `foo` from **_gremlins_** to **_system_**.

= ##### Packages can only be moved between repos **_only_** after they've been built. If a build has failed, you can't move the package to another repo, because there isn't one (package).

= ###### Package 'foo2' (already in **_community/galaxy_**) has been updated, as indicated by `buildtree -scu`. We must import the updates into the Artix trunk, edit the source files if needed and push the updates to the build server (again, `-s trunk` can be omitted):

    buildtree -i -p foo2
    (edit $workspace_dir/artix/packages-galaxy/foo2/trunk/PKGBUILD or any other source files inside trunk)
    communitypkg -p foo2 -s trunk -u

###### Move packages 'foo2' from repos/staging to repos/testing:

    testingpkg -p foo2 -s staging -u

###### Release package 'foo3' from trunk to repos/community (yes, `-s trunk` can be omitted):

    communitypkg -p foo3 -s trunk -u

###### Release package 'foo4' from trunk to repos/multilib-testing and push (in case you're still wondering, `-s trunk` can be omitted):

    multilib-testingpkg -p foo4 -s trunk -u

#### Suppose you've accidentally pushed **_multilib_**/foo4 into **_community_** with `communitypkg -p foo4 -u` and want to fix the screw-up. The build will fail in the first place, because of missing lib32 depends and there won't be any built package to move anyway. This is where we need a direct call of commitpkg, because the symlinks will not perform `-r` to remove a package from a repo.

    commitpkg -p foo4 -s community -ru     # To remove foo4 from community
    multilibpkg -p foo4 -u                 # To push foo4 into multilib

#### Use one `commitpkg` symlink operation at a time, i.e. do not edit both ``foo`` and ``bar`` and then `communitypkg -p foo -u`; `commitpkg` will complain about dirty workspace and rebase.

##### All packages in **_system_** **must** go through **[gremlins]** first! Only exception is the mirrorlists.

##### The jenkins pipeline (in the build server) will not trigger any action on the server, if only trunk has been modified, commited and pushed. To trigger a build the change **must** be copied over to _repos_ with the appropriate `commitpkg` symlink (e.g. `corepkg`) and the PKGBUILD pkgver must match the one in trunk, because that's where the pipeline gets the package version from.

* Jenkinsfile stages
    * Checkout
    * Build
    * Add
    * Remove

* Any repo moving operation of a package in git will trigger Add and Remove stages.
* Any new repos subdirectory will trigger the Build stage.
* The trunk dir will only trigger a checkout, modifications won't be built.

### Contributors

If you want a new or updated package merged into the git repository, please send a Pull Request with only the trunk of a given package. Please do not send PRs containing repos/$repo folders.
