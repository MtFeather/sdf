# 2019 1月份 SA@Tainan 1/20(日) GitLab 與 Git LFS 介紹
簡報:
- [git-lfs](http://gitlfs.rsync.tw)
- [.dev](http://gandi-dev.rsync.tw)
# demo
當被加入lfs檔案會將大檔案宣告為binary檔案，要使用lfs上傳
```bash
touch .gitattributes
git lfs track test.psd
```