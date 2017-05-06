
```shell
git checkout base-branch
glg --merges --grep=it-mica --pretty=oneline
git revert -m 1 <merge-commit>
```