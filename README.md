# homework
面试Demo
### 本方案因为时间原因使用的是Java后端开发中比较敏捷的SpringBoot进行开发，后端数据库使用的是Mysql。主要的实现思路是通过MD5加密长链接，产生一个新的32位的随机密码，然后通过截取字符串和位运算等一系列操作产生一个指定位数的短链接。访问转发的思路是通过Mysql数据库存放了长短链接，当一个短链接访问进入的时候，通过查询找到对应的长链接，返回给前台然后进行访问。通过读取配置文件的方式进行对应短链接，字符集的配置。使用Spring的junit进行了单元测试，并使用了RestClient进行了Http请求测试。
### 配置文件概览
```
##db配置
spring.atasource.driver-class-name= com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/homework?useUnicode=true&amp;characterEncoding=UTF-8
spring.datasource.username=root
spring.datasource.password=12345
```
##### 短链接配置支持用户自定义key 字符集 短链接长度 eg：
```
key=zhaoyang
chars=adasdasdasdasdasdqwewqe12312312asdasdsadazxczxcxz
strLen=6
```
#### 接口的具体实现方法如下
```
public ServerResponse getShortUrl(String longUrl) {

        Url url = urlDao.findByLongUrl(longUrl);
        if (url == null){
            ServerResponse.createSuccess("Success",url.getShortUrl());
        }
        String shortUrl = null;
        //判断生成的短链接数据库中是否存在
        while(true){
            shortUrl = encodeLongUrl(longUrl);
            url = urlDao.findByShortUrl(shortUrl);
            if (url == null) break;
        }
        //保存到数据库
        url = new Url();
        url.setLongUrl(longUrl);
        url.setShortUrl(shortUrl);
        urlDao.save(url);
        return ServerResponse.createSuccess("Success",shortUrl);
    }
```


#### 长链接生成短链接的函数
```
private String encodeLongUrl(String longUrl){
        //获取配置
        String key = PropertiesUtil.getProperty("key");
        String chars = PropertiesUtil.getProperty("chars");
        int intLen = Integer.parseInt(PropertiesUtil.getProperty("strLen"));
        char[] charArray = chars.toCharArray();
        //MD5对原链接进行加密
        String sMD5EncryptResult = DigestUtils.md5Hex(key + longUrl);
        //随机算法截取字符串
        String simpleStr = sMD5EncryptResult.substring(8,16);
        long lHexLong = 0x3FFFFFFF & Long.parseLong(simpleStr, 16);
        String resChars = "";
        for (int j = 0;j < intLen;j++){
            // 把得到的值与 0x0000003D 进行位与运算，取得字符数组 chars 索引
            long index = 0x0000003D & lHexLong;
            // 把取得的字符相加
            resChars += charArray[(int) index];
            // 每次循环按位右移 5 位
            lHexLong = lHexLong >> 5;
        }
        return resChars;
    }
```

#### 当访问一个短链接的时候 通过数据库的存储来查找到这个长链接
```
    @Override
    public ServerResponse getLongUrl(String shortUrl) {
    //判断长连接是否存在
        Url url = urlDao.findByShortUrl(shortUrl);
        if (url == null){
            return ServerResponse.createError("长链接不存在！！");
        }
        return ServerResponse.createSuccess("success",url.getLongUrl());
    }
```
### 访问计数 因不清楚是针对单个链接的访问次数还是服务器的访问次数 我这只做了对于数据库访问次数的记录，如需对单个链接访问次数进行记录，则还需一张Table来进行记录 不难实现。
```
package com.example.demo.controller;

import com.example.demo.response.ServerResponse;
import com.example.demo.service.IUrlService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

/**
 * @author zy
 * @create 2018-06-21 19:20
 **/
@Controller
public class TestMvcController {
    private  int num= 0;
    @Autowired
    private IUrlService iUrlService;

    @RequestMapping("/get_short_url")
    @ResponseBody
    public ServerResponse getShortUrl(@RequestParam("longUrl") String longUrl){
        addNum();
        if (longUrl == null || longUrl.equals("")){
            return ServerResponse.createError("域名错误！！！");
        }
        System.out.println(num);
        return iUrlService.getShortUrl(longUrl);
    }

    @RequestMapping("/get_long_url")
    @ResponseBody
    public ServerResponse getLongUrl(@RequestParam("shortUrl") String shortUrl){
        if (shortUrl == null || shortUrl.equals("")){
            return ServerResponse.createError("域名错误！！！");
        }
        addNum();
        System.out.println(num);
        return iUrlService.getLongUrl(shortUrl);
    }
    /**
    *增加访问次数 同步
    **/
    private synchronized void addNum(){
        num = num++;
    }
}

```

### 若需改变key的话 手动修改第二个配置就ok了 若需做成web端输入 也不难实现

### 单元测试 因时间原因 仅对 service层进行了单元测试，对整个流程使用RestClient进行了Http接口测试

```
package com.example.demo.service.serviceimpl;

import com.example.demo.response.ServerResponse;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import static org.junit.Assert.*;

/**
 * Created by 15520 on 2018/6/22.
 */
@SpringBootTest
@RunWith(SpringRunner.class)
public class UrlServiceimplTest {
    @Autowired
    UrlServiceimpl urlServiceimpl;
    @Test
    public void testGetShortUrl() throws Exception {
            ServerResponse serverResponse = urlServiceimpl.getShortUrl("http://www.xxxx.adasdsa/adasdas/asdasdsa/asdasdsadsa");
            System.out.print("url:"+serverResponse.getData());
    }

    @Test
    public void testGetLongUrl() throws Exception {
            ServerResponse serverResponse = urlServiceimpl.getLongUrl("http://127.0.0.1/1zzqzq");
            System.out.print(serverResponse.getData());
    }
}
```

#### 此demo 使用springboot 共分三层 Controller+Service+Dao 典型的三层架构 Service 与 dao 使用接口实现 Orm框架使用的是SpringDataJPA

#### 数据库sql如下 (因时间原因 未作数据库优化)
```
/*
Navicat MySQL Data Transfer

Source Server         : localhost_3306
Source Server Version : 50717
Source Host           : localhost:3306
Source Database       : homework

Target Server Type    : MYSQL
Target Server Version : 50717
File Encoding         : 65001

Date: 2018-06-22 01:02:21
*/

SET FOREIGN_KEY_CHECKS=0;

-- ----------------------------
-- Table structure for t_url
-- ----------------------------
DROP TABLE IF EXISTS `t_url`;
CREATE TABLE `t_url` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `long_url` varchar(512) DEFAULT NULL,
  `short_url` varchar(512) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of t_url
-- ----------------------------
INSERT INTO `t_url` VALUES ('1', 'http://127.0.0.1:8080/get_short_urlasdasdaszxasdasdasdasdsa', '13zzda');
INSERT INTO `t_url` VALUES ('2', 'http://127.0.0.1:8080/get_short_urlasdasdaszxasdasdasdasdsa', '13zzda');
INSERT INTO `t_url` VALUES ('3', 'http://www.xxxx.adasdsa/adasdas/asdasdsa/asdasdsadsa', 'http://127.0.0.1/1zzqzq');

```


