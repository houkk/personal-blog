# personal-blog
1. Create repo yourname.github.io
2. Configure _config.yml deply：
```

   deploy:
     type: git
     branch: master
     repo: {your github.io repo}
```
3. hexo clean
4. hexo generate -g
5. hexo deploy -g
