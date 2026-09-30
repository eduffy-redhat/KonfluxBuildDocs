# _I have a code change I want to turn into an alpha build!_

### Start here

First we need to get a PR ready for one of the 4 bundles. Lets take `rhdr-hub-operator-bundle` for example. We need to complete these steps:

1. Create your build of the ramen base image. See the base image repo for more info on this step.
2. Add a reference to that new image in the bundle repo and rendered the bundles how you need
    a. In the case of the hub bundle, you can add the image to `bundle-hack/update_bundle.sh` and run it which should modify some files in `bundle/`. **Don't commit these changes yet**.
3. Run `./makeAlphaRelease` which will convert this rendered bundle into an alpha release with a name like `0.0.0-<todays date>-<random hash>.alpha`. This will make several modifications to the files in the repo. **Now its time to commit these changes**, make sure you create a new branch so we can make a MR in gitlab.
    a. Its ok to push multiple alpha builds to the same MR branch, we just need the MR so it will trigger the pipeline on each push to the branch.
4. Push the new branch. Click the link to create a new MR in gitlab. The pipeline will start. Once its finished, copy down the image digest from Konflux. This is the alpha build image that we will use in the FBC repo

### Next, lets pivot to the [FBC repo](https://gitlab.cee.redhat.com/rh-ocp-dr/rhdr-catalog).

1. Take the image digest you copied from Konflux and paste it into `bundle-images.env` replacing the digest of its respective image.
2. Run `./scripts/create-alpha-release.sh`. This will pull each image in `bundle-images.env` and run it through `opm render`. It will automatically detect the CSV name and will make a matching bundle version (under `v4.22/catalog/<component>/bundles`), and add that bundle version to the alpha channel file (`v4.22/catalog/<component>/channels/alpha.json`)
    a. One thing to note, it will also check to see if a build is already in the alpha channel, if it is then it skips adding the duplicate entry. If you want to add an alpha that you've already added, just rename it (which needs to be done in the bundle repo by repeating the bundle steps before running this script)
3. Next, `git add ./v4.22 bundle-images.env` to add the modified and new files. Commit these to a new branch, push, and click the link to create a new MR in gitlab, this time on the FBC repo. The pipeline will be triggered
    a. Again, multiple alpha builds can go into one branch (in fact, its required they go into the same branch on the FBC repo to maintain an upgrade path that openshift will understand if you're pushing multiple alpha builds, see step 7 for more info)
4. Once the pipeline is completed, copy the full image URL and either create a new CatalogSource or update an existing one on your cluster setting spec.image to the copied image URL.
    a. Note: the bundle image we just built in the above section (and the base-image as well) is still only stored in the `redhat-user-workloads` registry, but the FBC rewrites the image URLs to the "production release" registry which, as of today, does not contain any of our images because we have no production releases yet. You will need an IDMS (ImageDigestMirrorSet) to point openshift to the correct location.
    b. Additionally, the IDMS we ship with the validated pattern points to our public quay mirror, so you'll need to copy the bundle and any other new images to the public quay mirror. Thankfully this can be done by running `./scripts/mirror-all-to-quay.sh` which will scan the repo for any images and copy them over to quay so the IDMS will be able to properly resolve the images
5. Check the catalog source is setup properly by running `oc get pods -n openshift-marketplace`. There should be a pod named after your catalog source. If that says `1/1 Ready` then we can proceed. If its in `ImagePullBackoff` or something similar, see 4.a and 4.b to make sure the image exists in the quay mirror and the IDMS is setup properly.
    a. If you're updating your operator, skip ahead to step 7.
6. Next we can install the new operator. Go to Ecosystem->Software Catalog and find the operator you want, make sure to pick the one with the CatalogSource name in the top right corner, there may be others from official repositories which will not contain your changes. Pick the "alpha" update channel in the dropdown and you should be able to see your alpha build(s) in the other dropdown.
7. If this isn't your first time installing the operator, it should automatically prepare to upgrade the operator assuming that the existing alpha version exists in the `alpha.json` and your new version has `replaces: <existing version>`. It may ask for approval depending on how you installed it the first time and if you've enabled automatic approvals.
    a. Its important to maintain a chain of alpha builds. If you instll build `abc123`, then create a new build `def456`, you need to make sure alpha.json says `def456` "replaces" `abc123`. Similarly, when you create build `ghi789`, `ghi789` replaces `def456`, so on and so on... This is handled automatically by `./scripts/create-alpha-release.sh`, but you'd need to keep adding builds to the same branch if you want them to keep the chain linked together. This allows openshift to determine the acceptable upgrade path.
8. After some time the old CSV will be uninstalled and the new version will start running. You can then check to make sure the operator is running the version you expect and run your tests against it.
