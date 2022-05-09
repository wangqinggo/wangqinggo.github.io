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