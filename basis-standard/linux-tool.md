## delete files 

```she
find ./ -type f -delete
```

[delete large amount of files](https://yonglhuang.com/rm-file/)

```she
find ./ -name "*.jar" -mtime +7 -size +30M -exec rm -rf {} \;
```

