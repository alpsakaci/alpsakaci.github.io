---
layout: post
title:  "Make Authenticated Request to Spotify"
date:   2021-10-17
categories: backend-development
tags: spring-boot oauth
---

### How to make authenticated request to Spotify from Spring Boot application?

To make an authenticated request to Spotify via Spring Boot you need to implement OAuth Client. If you don't know how to integrate Spotify OAuth client, you can read my previous [post][spotifyOauthClientPost].

In this chapter I will show you how to build an API Binding and add a Token Interceptor to Rest Template.

---

### Rest Template

To bring client-side capabilities to your application you can use Rest Template. *[The RestTemplate is the central Spring class for client-side HTTP access.][restTemplateDoc]* We will make requests to Spotify using Rest Template. The important part is these requests have to contain access token that provided by Spotify OAuth.

Our first step is building an abstract Api Binding that allows us to add token interceptor to RestTemplate.

---

### Building Abstract ApiBinding

The first step is building an ApiBinding class. This class has a RestTemplate field that reponsible for HTTP access.
We will use the Abstract ApiBinding to provide RestTemplate to classes by extending it.

Create a constructor that takes access token as a parameter. The access token will be added to HTTP request header as a bearer token.
To add bearer token to header use *getBearerTokenInterceptor* method. This method returns a ClientHttpRequestInterceptor object. The interceptors run before the HTTP request process, so our bearer token will be sent to server in every request which made with this restTemplate object. 

*If the access token is null, *getNoTokenInterceptor* method is executed and that throws an exception.*

{% highlight java %}
public abstract class ApiBinding {

  protected RestTemplate restTemplate;

  public ApiBinding(String accessToken) {
    this.restTemplate = new RestTemplate();
    if (accessToken != null) {
      this.restTemplate.getInterceptors().add(getBearerTokenInterceptor(accessToken));
    } else {
    this.restTemplate.getInterceptors().add(getNoTokenInterceptor());
    }
  }

  private ClientHttpRequestInterceptor getBearerTokenInterceptor(String accessToken) {
    ClientHttpRequestInterceptor interceptor = new ClientHttpRequestInterceptor() {
      @Override
      public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        request.getHeaders().add("Authorization", "Bearer " + accessToken);
        return execution.execute(request, body);
      }
    };

    return interceptor;
  }

  private ClientHttpRequestInterceptor getNoTokenInterceptor() {
    return new ClientHttpRequestInterceptor() {
      @Override
      public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        throw new IllegalStateException("Can't access the API without an access token");
      }
    };
  }
}
{% endhighlight %}

---

### Spotify API Service

Create a service class that extends ApiBinding. SpotifyApiService takes the access token and passes it to super class to build 
restTemplate. In our example we have a simple method that creates a playlist by given name. You can add more methods to use the [Spotify API][spotifyApi].

{% highlight java %}
public class SpotifyApiService extends ApiBinding {

  public SpotifyApiService(String accessToken) {
    super(accessToken);
  }

  public Object createPlaylist(String name) {
    Map<String, String> playlist = new HashMap<String, String>();
    playlist.put("name", name);

    Object response = this.restTemplate.postForObject("https://api.spotify.com/v1/me/playlists", playlist, Object.class);

    return response;
  }

}
{% endhighlight %}

---

### Passing Access Token to Service

In this step we will create a configuration class responsible for instantiating SpotifyApiService with access token that provided from OAuth Client Service. The method gets the token value if the client registiration ID equals 'spotify' and returns the SpotifyApiService. This configuration provide Autowire capability to SpotifyApiService.


{% highlight java %}
@Configuration
public class ApiConfig {
	
  @Bean
  @RequestScope
  public SpotifyApiService spotifyApiService(OAuth2AuthorizedClientService clientService) {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    String accessToken = null;
    if (authentication.getClass().isAssignableFrom(OAuth2AuthenticationToken.class)) {
      OAuth2AuthenticationToken oauthToken = (OAuth2AuthenticationToken) authentication;
      String clientRegistrationId = oauthToken.getAuthorizedClientRegistrationId();
      if (clientRegistrationId.equals("spotify")) {
        OAuth2AuthorizedClient client = clientService.loadAuthorizedClient(clientRegistrationId, oauthToken.getName());
        accessToken = client.getAccessToken().getTokenValue();
      }
    }
    return new SpotifyApiService(accessToken);
  }

}
{% endhighlight %}

---

### Controller

Create a controller to test the service. The *Autowired* annotation handles service instantiating, according to ApiConfig class.
Call the createPlaylist by name parameter to use the SpotifyApiService.

{% highlight java%}
@RestController
public class SpotifyContoller {

  @Autowired
  private SpotifyApiService apiService;

  @GetMapping("/createPlaylist")
  public Object createPlaylist(@RequestParam("name") String name) {
    return apiService.createPlaylist(name);
  }

}
{% endhighlight %}

---

### Conclusion

Make a GET request to /createPlaylist endpoint with 'name' parameter.

*e.g. /createPlaylist?name=Hello!*
![Create Playlist Request](/assets/article_images/2021-10-17-spotify-authenticated-request/1.png)

The playlist is created!
![Created Playlist](/assets/article_images/2021-10-17-spotify-authenticated-request/2.png)

---

*Note that this article aims to give you the key concept of making an authenticated request with RestTemplate. Make sure you use the appropriate data structures and HTTP methods while applying it to your project. (for example, using custom models instead of the Object class.)*

You can access the sample project on [GitHub][projectSource].

Thanks for reading.

[spotifyOauthClientPost]: https://alpsakaci.com/backend-development/2021/05/14/spotify-oauth.html
[restTemplateDoc]: https://spring.io/blog/2009/03/27/rest-in-spring-3-resttemplate/
[spotifyApi]: https://developer.spotify.com/documentation/web-api/
[projectSource]:      https://github.com/alpsakaci/spotify-oauth-sample