---
title: Spring Security6 + React(Thymeleaf以外) + JWT(JSON Web Token)
tags:
  - JWT
  - React
  - SpringBoot
  - axios
  - SpringSecurity6
private: false
updated_at: '2024-12-15T03:33:37+09:00'
id: e3b43cdfadfb24266c32
organization_url_name: null
slide: false
ignorePublish: false
---

こんにちは、hidepon4649 です。
ポートフォリオ作成中に Spring Security6 + React + JWT でハマったのでメモします。

# 発生したエラー

```エラーログ
Full authentication is required to access this resource
```

リクエストを跨いだ JWT トークンの引継が出来ていないようです。

# 構成

デバッグの前後で構成は変えていません。変更したのはソースコードのみ。

```npm.list
├── @emotion/react@11.13.5
├── @emotion/styled@11.13.5
├── @mui/icons-material@6.1.10
├── @mui/material@6.1.10
├── @types/jest@29.5.14
├── @types/node@22.10.1
├── @types/react-dom@18.3.2
├── @types/react@18.3.14
├── axios@0.21.4
├── bootstrap@5.3.3
├── react-dom@18.3.1
├── react-router-dom@6.28.0
├── react-scripts@5.0.1
├── react@18.3.1
└── typescript@4.9.5
```

```build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.4.0'
    // 〜 中略 〜
}
// 〜 中略 〜
java {
  toolchain {
    languageVersion = JavaLanguageVersion.of(17)
  }
}
// 〜 中略 〜
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-security:3.4.0' // セキュリティ関連
  implementation 'org.springframework.security:spring-security-config:6.4.1' // セキュリティ関連
  implementation 'org.springframework.security:spring-security-web:6.4.1' // セキュリティ関連
  implementation 'io.jsonwebtoken:jjwt-api:0.12.6'  // JWT関連
  runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.6'    // JWT関連
  runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.6' // JWT関連
  // 〜 中略 〜
}
// 〜 中略 〜
```

```application.properties
# JWTの設定 (app.jwtSecretの値はアレンジして下さい)
app.jwtSecret=mySecretKey0123456789NaishoNaishoNaishoNaishoNaishoNaishoNaisho999990
app.jwtExpirationMs=3600000
```

# デバッグ対応

対応内容

- 1 .ログイン認証(ユーザテーブル問い合わせ)に成功したら、JWT トークンを発行する
- 2 .以降、Http リクエストする際に必ず実施する事が 1 点
  - 2-1 . 上記で発行された JWT トークンをリクエストヘッダーに設定
- 以下がデバッグ対応後の実装です。

## React

```LoginPage.tsx
import React, { useState } from "react";
import axios from "axios";
import { useNavigate } from "react-router-dom";

const LoginPage = () => {

  // 〜 中略 〜

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {

      // 〜 中略 〜

      // 認証リクエスト
      const response = await axios.post(
        "http://localhost:8080/api/auth/login",
        { email, password }
      );

      // JWTトークンを保持
      localStorage.setItem("token", response.data.token);

      // トップページに遷移
      navigate("/任意のトップページ");

    } catch (error) {
      // エラー処理
    }
  };

  return (
    <div className="mx-3 mt-3">
      <h2>ログイン</h2>
      <form onSubmit={handleSubmit}>
      // 〜 中略 〜
      </form>
    </div>
  );
};

export default LoginPage;
```

```以降のリクエスト.tsx

// 認証時に発行されたJWTトークンを利用
const token = localStorage.getItem("token");

const response = await axios.get(
  `http://localhost:8080/パス`,
  {
    // リクエストヘッダにJWTトークンを載せる
    headers: {
      Authorization: `Bearer ${token}`,
    },
  }
);

```

## Java

```SecurityConfig.java
// 〜 中略 〜
import com.example.attendancemanager.security.JwtAuthEntryPoint;
import com.example.attendancemanager.security.JwtAuthTokenFilter;
import com.example.attendancemanager.service.CustomUserDetailsService;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthEntryPoint jwtAuthEntoryPoint;
    private final JwtAuthTokenFilter jwtAuthenticationFilter;
    private final CustomUserDetailsService userDetailsService;

    public SecurityConfig(JwtAuthTokenFilter jwtAuthenticationFilter,
      CustomUserDetailsService userDetailsService, JwtAuthEntryPoint jwtAuthEntoryPoint) {
      this.jwtAuthenticationFilter = jwtAuthenticationFilter;
      this.userDetailsService = userDetailsService;
      this.jwtAuthEntoryPoint = jwtAuthEntoryPoint;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
      http
        // 〜 中略 〜
        .authorizeHttpRequests(authorize -> authorize
          .requestMatchers("/api/auth/**").permitAll() // JWTトークン発行前のログイン認証
          .anyRequest().authenticated())
        .exceptionHandling(exception -> exception.authenticationEntryPoint(jwtAuthEntoryPoint)) // JWT関連
        .addFilterBefore(jwtAuthenticationFilter,UsernamePasswordAuthenticationFilter.class); // JWT関連

      return http.build();
    }

    // 〜 中略 〜
}

```

```AuthController.java
// 〜 中略 〜
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
// 〜 中略 〜
import com.example.attendancemanager.security.JwtRequest;
import com.example.attendancemanager.security.JwtResponse;
import com.example.attendancemanager.security.JwtUtils;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

  @Autowired
  private AuthenticationManager authenticationManager;

  @Autowired
  private JwtUtils jwtUtils;

  @PostMapping("/login")
  public ResponseEntity<?> authenticateUser(@RequestBody JwtRequest jwtRequest) {
    Authentication authentication;
    try {
      // 認証トライ
      authentication = authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(jwtRequest.getEmail(), jwtRequest.getPassword()));

    } catch (AuthenticationException e) {
      // 認証失敗
      Map<String, Object> map = new HashMap<>();
      map.put("message", "Bad credentials");
      map.put("status", false);
      return new ResponseEntity<Object>(map, HttpStatus.NOT_FOUND);
    }

    // 認証成功時、JWTトークンを発行
    SecurityContextHolder.getContext().setAuthentication(authentication);
    UserDetails userDetails = (UserDetails) authentication.getPrincipal();
    String jwtToken = jwtUtils.generateToken(userDetails.getUsername());
    List<String> roles = userDetails.getAuthorities().stream()
      .map(item -> item.getAuthority())
      .collect(Collectors.toList());

    JwtResponse response = new JwtResponse(userDetails.getUsername(), roles, jwtToken);
    return ResponseEntity.ok(response);

  }

  // 〜 中略 〜

}


```

```JwtUtils.java
// 〜 中略 〜
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.MalformedJwtException;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.ExpiredJwtException;
import io.jsonwebtoken.UnsupportedJwtException;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;
import java.security.Key;
import java.util.Date;
// 〜 中略 〜

@Component
public class JwtUtils {

  private static final Logger logger = LoggerFactory.getLogger(JwtUtils.class);

  @Value("${app.jwtSecret}")
  private String jwtSecret;

  @Value("${app.jwtExpirationMs}")
  private int jwtExpirationMs;

  private Key key() {
    return Keys.hmacShaKeyFor(Decoders.BASE64.decode(jwtSecret));
  }

  public String generateToken(String email) {
    return Jwts.builder()
      .subject(email)
      .issuedAt(new Date())
      .expiration(new Date((new Date()).getTime() + jwtExpirationMs))
      .signWith(key())
      .compact();
  }

  public boolean validateToken(String token) {
    try {
      Jwts.parser().verifyWith((SecretKey) key()).build().parseSignedClaims(token);
      return true;
    } catch (MalformedJwtException e) {
      logger.error("Invalid JWT token: {}", e.getMessage());
    } catch (ExpiredJwtException e) {
      logger.error("JWT token is expired: {}", e.getMessage());
    } catch (UnsupportedJwtException e) {
      logger.error("JWT token is unsupported: {}", e.getMessage());
    } catch (IllegalArgumentException e) {
      logger.error("JWT claims string is empty: {}", e.getMessage());
    }
    return false;

  }

  public String getEmailFromToken(String token) {
    Claims claims = Jwts.parser()
      .verifyWith((SecretKey) key())
      .build().parseSignedClaims(token)
      .getPayload();

    return claims.getSubject();
  }

}
```

```JwtRequest.java
public class JwtRequest {

  private String email;
  private String password;

  // アクセサメソッド
  // 〜 中略 〜
}

```

```JwtResponse.java
import java.util.List;

public class JwtResponse {

  private String token;
  private String username;
  private List<String> roles;

  public JwtResponse(String username, List<String> roles, String token) {
      this.username = username;
      this.roles = roles;
      this.token = token;
  }
  // アクセサメソッド
  // 〜 中略 〜

}

```

```JwtAuthTokenFilter.java
import java.io.IOException;

import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class JwtAuthTokenFilter extends OncePerRequestFilter {

  private static final Logger logger = LoggerFactory.getLogger(JwtAuthTokenFilter.class);

  private final JwtUtils jwtUtils;
  private final UserDetailsService userDetailsService;

  public JwtAuthTokenFilter(JwtUtils jwtUtils, UserDetailsService userDetailsService) {
    this.jwtUtils = jwtUtils;
    this.userDetailsService = userDetailsService;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
    throws ServletException, IOException {

    String token = parseToken(request);
    if (token != null && jwtUtils.validateToken(token)) {
      String email = jwtUtils.getEmailFromToken(token);
      UserDetails userDetails = userDetailsService.loadUserByUsername(email);
      UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
        userDetails, null, userDetails.getAuthorities());

      logger.debug("Roles from JWT: {}", userDetails.getAuthorities());

      authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

      SecurityContextHolder.getContext().setAuthentication(authentication);
    }

    chain.doFilter(request, response);

  }

  private String parseToken(HttpServletRequest request) {
    String bearerToken = request.getHeader("Authorization");
    logger.debug("Authorization Header: {} ", bearerToken);
    if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
        return bearerToken.substring(7);
    }
    return null;
  }

}

```

```JwtAuthEntryPoint.java
// 〜 中略 〜
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import com.fasterxml.jackson.databind.ObjectMapper;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class JwtAuthEntryPoint implements AuthenticationEntryPoint {

  private static final Logger logger = LoggerFactory.getLogger(JwtAuthEntryPoint.class);

  @Override
  public void commence(HttpServletRequest request, HttpServletResponse response,
          AuthenticationException authException) throws IOException, ServletException {

    logger.error("Unauthorized error: {}", authException.getMessage());

    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

    final Map<String, Object> body = new HashMap<>();
    body.put("status", HttpServletResponse.SC_UNAUTHORIZED);
    body.put("error", "Unauthorized");
    body.put("message", authException.getMessage());
    body.put("path", request.getServletPath());

    final ObjectMapper mapper = new ObjectMapper();
    mapper.writeValue(response.getOutputStream(), body);

  }

}

```

以上

私が上述のエラーでハマった際に、
Spring Security6 + React + JWT の日本語情報がなかなか見つけられなくて無駄に時間を費やしてしまいました。誰かのお役に立てれば幸いです。(私の検索能力が低いだけかも‥？)
