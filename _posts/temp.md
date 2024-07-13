基于Go & React 构建一个网络下载收藏歌曲的网站 


功能：
- 用户注册登录，基于邮箱
- 根据音乐名搜索，去网站（bilibili,youtube...）找对应视频，转化为音频，音频可以在线试听、下载、收藏、加入歌单
- 用户可以创建歌单，下载音乐，分享


单体架构

语言go
web框架: gin
ORM框架：gorm
email: gomail
job: xxl-job
分布式锁：etcd
数据库：mysql
音乐存储：对象存储

前端：React
