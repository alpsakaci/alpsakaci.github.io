---
layout: post
title:  "Spotify OAuth Client with Spring Boot"
date:   2021-05-14
categories: backend-development
# tags: featured
# image: /assets/article_images/2021-05-14-spotify-oauth/x.jpg
# image2: /assets/article_images/2021-05-14-spotify-oauth/x.jpg
---

## How to Integrate Spotify OAuth to Spring Boot Application?

Spring boot has an OAuth2 Login feature to allow users to log in to your application with their existing account. You can integrate OAuth 2.0 providers to Spring Boot application with a few configuration. If you read the Spring Boot documentation you will notice that Spring Boot includes pre-defined configurations for well known providers such as Google and Facebook. To integrate your application to Spotify OAuth provider you have to add custom configurations.
<!-- link -->

---
### Step 1: Initialize a demo app

First of all, we will create an app using Spring initializr. Go to [Spring initializr][springinit] to generate an app with Spring Web and OAuth2 Client dependencies.

![Init Spring Boot App](/assets/article_images/2021-05-14-spotify-oauth/1.png)

Make sure that you have these dependencies in your pom.xml
{% highlight xml%}
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
{% endhighlight %}

---
### Step 2: Create an app on Spotify Developer

To use the Spotify API you have to register your app.
- Go to  [Spotify Developer Dashboard][spotifydeveloperdashboard] and create an app.
- Add your Website and Redirect URIs under the app settings.

![Spotify App Settings](/assets/article_images/2021-05-14-spotify-oauth/2.png)

The default Redirect URI of Spring OAuth is {base_url}/login/oauth2/code/{provider}.

We will use Client ID and Client Secret on Spring Boot app.

---

### Step 3: Configuring Spring Security

- Create a class with @Configuration annotation that extends WebSecurityConfigurerAdapter.
- Override configure method to secure endpoints.

{% highlight java %}
@Configuration
public class OAuth2LoginSecurityConfig extends WebSecurityConfigurerAdapter {
	
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		// @formatter:off
		http
			.authorizeRequests(authorize -> authorize.anyRequest().authenticated())
			.oauth2Login();
		// @formatter:on
	}
	
}
{% endhighlight %}

- Configuration class must return a bean.

Define a bean that returns ClientRegistrationRepository. In this sample project we will use an In Memory Repository.

{% highlight java %}
@Bean
public ClientRegistrationRepository clientRegistrationRepository() {
    return new InMemoryClientRegistrationRepository(this.spotifyClientRegistration());
}
{% endhighlight %}

- Configure Client Registration

We will register our client to Spring Security. OAuth Client library will handle authentication with this configuration. Register the client with your Client ID and Client Secret. Make sure that Redirect URI format is identical to Spotify settings. The user object returned from Spotify API includes user name with "display_name" attribute name. You have to set this attribute name so that Spring Security can register the user name.

{% highlight java %}
private ClientRegistration spotifyClientRegistration() {
    // @formatter:off
    return ClientRegistration
            .withRegistrationId("spotify")
            .clientId("your-client-id")
            .clientSecret("your-client-secret")
            .clientAuthenticationMethod(ClientAuthenticationMethod.BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
            .scope("user-read-private", "user-read-email")
            .authorizationUri("https://accounts.spotify.com/authorize")
            .tokenUri("https://accounts.spotify.com/api/token")
            .userInfoUri("https://api.spotify.com/v1/me")
            .userNameAttributeName("display_name")
            .clientName("spotify")
            .build();
    // @formatter:on
}
{% endhighlight %}

- The Complete Class

{% highlight java %}
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.oauth2.client.registration.ClientRegistration;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.client.registration.InMemoryClientRegistrationRepository;
import org.springframework.security.oauth2.core.AuthorizationGrantType;
import org.springframework.security.oauth2.core.ClientAuthenticationMethod;

@Configuration
public class OAuth2LoginSecurityConfig extends WebSecurityConfigurerAdapter {
	
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		// @formatter:off
		http
			.authorizeRequests(authorize -> authorize.anyRequest().authenticated())
			.oauth2Login();
		// @formatter:on
	}

	@Bean
	public ClientRegistrationRepository clientRegistrationRepository() {
		return new InMemoryClientRegistrationRepository(this.spotifyClientRegistration());
	}

	private ClientRegistration spotifyClientRegistration() {
		// @formatter:off
		return ClientRegistration
				.withRegistrationId("spotify")
				.clientId("your-client-id")
				.clientSecret("your-client-secret")
				.clientAuthenticationMethod(ClientAuthenticationMethod.BASIC)
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
				.scope("user-read-private", "user-read-email")
				.authorizationUri("https://accounts.spotify.com/authorize")
				.tokenUri("https://accounts.spotify.com/api/token")
				.userInfoUri("https://api.spotify.com/v1/me")
				.userNameAttributeName("display_name")
				.clientName("spotify")
				.build();
		// @formatter:on
	}
	
}

{% endhighlight %}

### Step 4: Create a Rest Controller

With this controller we will access the user name that authenticated with OAuth.

{% highlight java %}
@RestController
public class UserController {
	
	@RequestMapping("/user")
	public Map<String, Object> user(@AuthenticationPrincipal OAuth2User principal) {
		return Collections.singletonMap("name", principal.getAttribute("display_name"));
	}

}
{% endhighlight %}

---

### Step 5: Create an index page to test secure endpoint

Create an index.html under src/main/resources/static

{% highlight html %}
<html>
<head>
</head>

<body>
Hello!
</body>

</html>

{% endhighlight %}

### That's it. Run your project and test the app.

Spring Security will redirect you to Spotify login page. After authenticated successfully, you can access index page and /user endpoint.

![Spotify Login Page](/assets/article_images/2021-05-14-spotify-oauth/3.png)

![Index Page](/assets/article_images/2021-05-14-spotify-oauth/4.png)

![user endpoint](/assets/article_images/2021-05-14-spotify-oauth/5.png)

You can access the sample project on [GitHub][projectSource]. In the next post, I will add an API binding feature to the project. This feature allows you to make authenticated request to Spotify API via your application.

Thanks for reading.

[springinit]:      https://start.spring.io
[spotifydeveloperdashboard]: https://developer.spotify.com/dashboard
[projectSource]:      https://github.com/alpsakaci/spotify-oauth-sample