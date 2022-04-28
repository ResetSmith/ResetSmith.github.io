Since the theme is a submodule, when updating it you must make the changes, and then tell GitHub to pull a new version of the submodule once the changes have been made. To do this open the project in CLI and do a
```
git pull
```
Then to tell the project to pull an updated version of the submodule
```
git submodule update --init --recursive --remote
```
Then push the changes into the repository.
