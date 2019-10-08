# 常见问题
## git clone 不下来，要什么 personal access token
问题：
```
$ git clone https://git.tsinghua.edu.cn/daizy19/decaf-java-19.git
Cloning into 'decaf-java-19'...
Username for 'https://git.tsinghua.edu.cn': x
Password for 'https://x@git.tsinghua.edu.cn':
remote: HTTP Basic: Access denied
remote: You must use a personal access token with 'api' scope for Git over HTTP.
remote: You can generate one at https://git.tsinghua.edu.cn/profile/personal_access_tokens
fatal: Authentication failed for 'https://git.tsinghua.edu.cn/daizy19/decaf-java-19.git/'
```

解决：
1. 进入自己 git.tsinghua 的 Settings，选择左侧的 Access Tokens，
    填写信息类似如下，点击 **Create personal access token**：
![](assets/git-1.png)

2. 在生成的页面中有你的 personal access token，如下图中的 `cnxKrqTDoyXhUzZBMEgG`：
![](assets/git-2.png)

3. 往下滚动，确认 Active Personal Access Tokens 中有刚刚的那个 `somenamebb`
![](assets/git-3.png)

4. 重新执行 git clone 命令，**你需要 clone 的 url 是 `git clone https://oauth2:令牌@git.tsinghua.edu.cn/compiler-19/decaf-学号.git`**，其中令牌是刚刚那个的 personal access token。
```
$ git clone https://oauth2:cnxKrqTDoyXhUzZBMEgG@git.tsinghua.edu.cn/daizy19/decaf-java-19.git
Cloning into 'decaf-java-19'...
remote: Counting objects: 1144, done.
remote: Compressing objects: 100% (507/507), done.
remote: Total 1144 (delta 620), reused 927 (delta 518)
Receiving objects: 100% (1144/1144), 276.05 KiB | 1.18 MiB/s, done.
Resolving deltas: 100% (620/620), done.
```
