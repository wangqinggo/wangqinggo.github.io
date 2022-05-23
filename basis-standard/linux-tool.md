## delete files 

```shell
find ./ -type f -delete
```

[delete large amount of files](https://yonglhuang.com/rm-file/)

```shell
find ./ -name "*.jar" -mtime +7 -size +30M -exec rm -rf {} \;
```

```shell
ls -alrst|head -n 30|awk '{print $10}'|xargs rm
```



# Clean up old git branches

```shell
# DANGER! Only run these if you are sure you want to delete unmerged branches.
# delete all local unmerged branches
git branch --no-merged | egrep -v "(^\*|main|test|dev)" | xargs git branch -D
# delete all local branches (merged and unmerged).
git branch | egrep -v "(^\*|main|test|dev)" | xargs git branch -D


# DANGER! Only run these if you are sure you want to delete unmerged branches.
# delete all remote unmerged branches
git branch -r --merged | egrep -v "(^\*|main|test|dev)" | sed 's/origin\///' | xargs -n 1 git push origin --delete
# delete all remote branches (merged and unmerged).
git branch -r | egrep -v "(^\*|main|dev|test)" | sed 's/origin\///' | xargs -n 1 git push origin --delete
```

