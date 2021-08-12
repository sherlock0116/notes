# 【Git】GitHub Fork后和源码同步更新方法

在github上fork别人的项目后，我们一般会clone到本地，然后进行阅读修改，那么我们如何与源仓库进行同步呢？

下面我们以最近火热的spring-cloud-alibaba为例讲述这一操作。



1. 登陆自己的github并fork spring-cloud-alibaba（https://github.com/spring-cloud-incubator/spring-cloud-alibaba），fork完成后效果如下

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbAicJao1DcI6Hsbw0icMbd8rCmVr1YGS0n2SFfFZ40RK5EhWW0nN2zb2Q/640?wx_fmt=png)



2. 复制自己github中spring-cloud-alibaba项目地址，使用git clone https://github.com/jianzh5/spring-cloud-alibaba(自己仓库)到本地

   ```sh
   git clone https://github.com/jianzh5/spring-cloud-alibaba
   ```

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbib8fjb5RZ0nfms7mK58IRAvFBsrFgkqqxFGlbfibVzRkRkN7k3WyAibdg/640?wx_fmt=png)



3. 使用git remote add upstream 建立源版本 upstream，即你 fork 的项目的源码地址

   ```sh
   git remote add upstream https://github.com/spring-cloud-incubator/spring-cloud-alibaba.git
   ```

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbzI45s95zREjpteup5gvUOfKk8WSjFGyfQE7FzHWBCVibGCNRrsUCY0w/640?wx_fmt=png)



4. 使用git remote -v 查看所有版本记录

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbjapDFxQL4TrZQgNvZwS6Le5vyhWUCQxm5W0D4UkNQkicyrUOzbkWUlA/640?wx_fmt=png)



5. 使用git fetch upstream 将源主机的更新全部取回本地

   ```sh
   git fetch upstream
   ```

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbEvj7UWIbxh641Y5UsHgTiavHdOsaLFY1NH9pA3uR9ibEsOIN3PfHzSfg/640?wx_fmt=png)



6. 使用 `git branch -a` 查看所有版本

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbOkMP7ZYvNGDmkjBvMTXJbXZ8vbI2RNayqyo1hyuGlchd8RE2CzC4dg/640?wx_fmt=png)



7. 将源主机更新与本地代码合并,此时需要指定版本，我们这里选择 master 版本

   ```sh
   git merge upstream/master
   ```

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbxic70H7yvFaODtx1e2Zu5Qiccyj3oqfow5JQ8mZyp3YH1jl31ewiaYs7w/640?wx_fmt=png)



8. 将合并后的代码提交到自己github上

   ```sh
   git add .	
   git commit -m  “Sync from upstream”	
   git push
   ```

   ![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/EnzAwvpTQJ7Z8PyhhK758aUt7oeknibicbVYljqaJicWUOKhoAyYSqtn07KDgxXfZoCevricqYM8KeUfUzJ61RLsog/640?wx_fmt=png)

经过这几步即可完成对代码的合并提交与更新，登陆github查看同步后的效果。