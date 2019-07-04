---
layout: post
title:  "携程配置中心apollo sso登录"
date:   2019-06-04 14:06:05
categories: JAVA APOLLO
excerpt: APOLLO一站式登录。
---

* content
{:toc}

## SSO登录步骤

* 登录内网平台
* 携带内网登录登录凭证token回调apollo的sso接口
* sso接口根据token请求业务接口获取基本信息
* 信息入库,供apollo分配权限
* 修改认证模式,写入session


---

### 注入一个bean

```
package com.ctrip.framework.apollo.portal.spi;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

### sso登录认证

```
package com.ctrip.framework.apollo.portal.controller;

import com.ctrip.framework.apollo.common.exception.BadRequestException;
import com.ctrip.framework.apollo.core.utils.StringUtils;
import com.ctrip.framework.apollo.portal.constant.LoginConstant;
import com.ctrip.framework.apollo.portal.entity.po.UserPO;
import com.ctrip.framework.apollo.portal.spi.LogoutHandler;
import com.ctrip.framework.apollo.portal.spi.UserInfoHolder;
import com.ctrip.framework.apollo.portal.spi.UserService;
import com.ctrip.framework.apollo.portal.spi.springsecurity.SpringSecurityUserService;
import com.ctrip.framework.apollo.portal.util.*;
import com.google.gson.Gson;
import org.apache.commons.httpclient.protocol.Protocol;
import org.apache.commons.httpclient.protocol.ProtocolSocketFactory;
import org.apache.http.HttpException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetails;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

/**
 * sso登录
 * @author yeping@mail.jj.cn
 * @version 1.0
 * @date 2019/5/24 9:48
 */
@RestController
@RequestMapping("/sso")
public class SsoController {
    private final UserInfoHolder userInfoHolder;
    private final LogoutHandler logoutHandler;
    private final UserService userService;

    @Autowired
    protected AuthenticationManager authenticationManager;

    @Autowired
    protected Gson gson;

    public SsoController(final UserInfoHolder userInfoHolder,
                         final LogoutHandler logoutHandler,
                         final UserService userService){
        this.userInfoHolder = userInfoHolder;
        this.logoutHandler = logoutHandler;
        this.userService = userService;

    }

    @GetMapping("/login")
    public Object login(HttpServletRequest request, HttpServletResponse response) throws HttpException,IOException {

        //省略通过token查出用户json格式信息的步骤
        
        String infoStr = "{........}";//json格式信息
        //infoMap 存储对应的登录人信息键值对

        Map<String,String> infoMap = gson.fromJson(infoStr,Map.class);
        if(infoMap == null || infoMap.size() == 0){
            throw new BadRequestException("error : esb info by token  trans error : " + infoStr);
        }

        //当前登录人实例
        UserPO userPO = new UserPO();
        String account = infoMap.get(LoginConstant.USER_ACOUNT);
        String defaultPass = account + LoginConstant.PSWD_SUFIX;
        userPO.setUsername(account);
        userPO.setPassword(defaultPass);
        userPO.setEmail(account + LoginConstant.EMAIL_SUFIX);

        if (userService instanceof SpringSecurityUserService) {
            //登录人信息入库
            ((SpringSecurityUserService) userService).createOrUpdate(userPO);

            //用户名 密码生成认证的token类型
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(account,defaultPass);
            //设置 authenticationToken 的details,初始化一些request信息
            authenticationToken.setDetails(new WebAuthenticationDetails(request));
            //使用authenticationManager接口中的authenticate进行SpringSecurity认证
            Authentication authenticationUser = authenticationManager.authenticate(authenticationToken);
            //设置当前认证对象
            SecurityContextHolder.getContext().setAuthentication(authenticationUser);
            //设置session
            request.getSession().setAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,SecurityContextHolder.getContext());

        } else {
            throw new UnsupportedOperationException("Create or update user operation is unsupported");
        }

        return null;
    }
}
```

---