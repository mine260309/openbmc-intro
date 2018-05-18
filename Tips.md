# Tips

In this article, it introduces various tips during building, developing and
testing OpenBMC.
It assumes that the reader already has experience of OpenBMC or Bitbake and
know the related concepts, e.g. recipes.

Note: refer to [cheatsheet.md][1] for existing tips. This article is expected
to be merged to that doc in future.

### devtool

`devtool` is a convenient utility in Yocto to make changes in local directory.
Typical usage is:
```
# To create a local copy of recipe's code and build with it
devtool modify <recipe>
cd build/workspace/sources/<recipe>  # And make changes
bitbake obmc-phosphor-image  # Build with local changes

# After you have finished, reset the recipe to ignore local changes
devtool reset <recipe>
```

To use this tool, you need the build environment, e.g. `. oe-init-build-env`.
It will add `<WORKDIR>/scripts/` to your PATH and `devtool` is in it.

Below are real examples.

#### devtool on ipmi
If you want to debug or adding new function in ipmi, you probably need to
change the code in [phosphor-host-ipmid][2].
Checking the recipes, you know this repo is in [phosphor-ipmi-host.bb][3].
So below are the steps to use devtool to modify the code locally, build and
test it.
1. Use devtool to create local repo
   ```
   devtool modify phosphor-ipmi-host
   ```
   It clones the repo into `build/workspace/sources/phosphor-ipmi-host`, create
   and checkout branch `devtool`.
2. Make changes in the repo. E.g. adding code to handle new ipmi commands, or
   simply adding trace logs. Say you have modified `apphandler.cpp`
3. Now you can build the whole image or the ipmi recipe it self.
   ```
   bitbake obmc-phosphor-image  # Build the whole image
   bitbake phosphor-ipmi-host  # Build the recipe
   ```
4. To test your change, either flash the whole image, or replace the changed
   binary. Note that the changed code is built into `libapphandler.so` and it
   is used by both host and net ipmi daemon.
   Here let's copy the changed binary to BMC because it is easier to test.
   ```
   # Replace libapphandler.so.0.0.0
   scp build/workspace/sources/phosphor-ipmi-host/oe-workdir/package/usr/lib/ipmid-providers/libapphandler.so.0.0.0 root@bmc:/usr/lib/ipmid-providers/
   systemctl restart phosphor-ipmi-host.service  # Restart the inband ipmi daemon
   # Or restart phosphor-ipmi-net.service if you want to test net ipmi
   ```
5. Now you can test your changes.



[1]: https://github.com/openbmc/docs/blob/master/cheatsheet.md
[2]: https://github.com/openbmc/phosphor-host-ipmid
[3]: https://github.com/openbmc/openbmc/blob/master/meta-phosphor/common/recipes-phosphor/ipmi/phosphor-ipmi-host.bb
