---
layout: post
title:  "Coverage is cheap, Can it catch bugs?"
date:   2024-04-16 06:21:52 -0400
categories: Testing
---

While unit testing especially while working with reflective based frameworks like spring and jacksonparser much of the behaviour is decided at runtime.

When we go ahead and aggressively mock a lot of behaviour u end up testing nothing. It looks like we have test coverage for the area.
But much of the behavior of reflective-based apps are decided at runtime.

Traditional forms of test coverage like line and branch coverage don't add much to testing value especially

in reflective environments.


{% highlight java %}
 @Test
    void testGetResult() {
        when(restTemplate.getForEntity(any(), any())).thenReturn(ResponseEntity.ok().body(pltResult));
        assertEquals(pltResult, client.getScanResult("1","https://abc/123"));
    }

{% endhighlight %}

When we mock restTemplate is this part of code u essentially test nothing. Things like parsing of objects and loading of the right resttempale and object mapper configuration are never tested

This is a classic case of mock everything and testing nothing. But we can show coverage that src part is touched by the test

{% highlight java %}

/**
     * @TestPlan Set up the request and response for the libraries:coordSearch with the a wire mock server
     * and check if the client is able call the stub api and parse data successfully
     */
    @Test
    public void testGetScanResult() throws Exception {
        setField(platformClient, "baseUrl", stubServer);

        final var scanId = "scanid";
        final var scanuri = "/scanurl1";

        final var responseFile = PlatformClientApiTest.class.getResource("/platform/getplatscan.json");
        final var response = Files.readString(Paths.get(responseFile.toURI()));

        final var uri = UriComponentsBuilder.fromUriString(scanuri).build().toUri();
        stubFor(get(urlEqualTo(uri.toString()))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withBody(response)
                        .withHeader("Content-Type", "application/json")
                )
        );

        final var actual = platformClient.getScanResult(scanId, stubServer + scanuri);

        assertEquals("project-1", actual.getProjectId());
        assertEquals("space-1", actual.getWorkspaceName());
        assertEquals("maven:coord1:coord2:1.2", actual.getLibraryInstances().get(0).getLibraryInstanceRef());
        assertEquals("SRCCLR-SID-1234", actual.getVulns().get(0).getSrcclrId());
    }

{% endhighlight %}


And use SpringbootTest which actually loads the config which would be loaded at runtime with appropriate TestProperties

{% highlight java %}
/**
 * Tests the full functionality of the PlatformClientApi by setting up the wire mock with actual request and response
 * anc check if the parsing works without issues.
 */
@TestPropertySource(locations = "classpath:/tests.properties")
@SpringBootTest(classes = {ClientConfigs.class, PlatformAuthInterceptor.class})
public class PlatformClientApiTest {
{% endhighlight %}


When we rewrite this form a test. Suddenly the unit test tests a lot more of things. A real server is simulated by a wire mock. And also you test the real resttemplate config which would get created while one is running the code. Any reflective-based framework behavior would be known in the runtime and not in the compile time. So the test should simulate the actual running state of the code.

Testing DB interfaces 

Another common mistake in test testing DB interfaces is to mock the JPA repository

 
{% highlight java %}
class LibraryInstanceServiceImplMethodsTest {

  ///////////////////////////// Class Attributes \\\\\\\\\\\\\\\\\\\\\\\\\\\\\\

  ////////////////////////////// Class Methods \\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\

  //////////////////////////////// Attributes \\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\

  @Mock
  private LibraryInstanceRepository instanceRepo;
{% endhighlight %}

 The above test doesn’t not capture any bug with the repo part of code as its mostly would reflectively buy the hibernate framework and doesn’t test the sql part also 

 
{% highlight java %}
@TestPropertySource(properties = {
    "spring.flyway.enabled=true",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
@DataJpaTest
@EnableJpaAuditing
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import({TestWithTestContainerConfig.class})
public class LibraryModifyServiceImplDataJpaTest {

  //////////////////////////////// Attributes \\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\

  @Autowired
  private LibraryInstanceRepository instanceRepo;
{% endhighlight %}

 The usage of data jpa test creates a full sql server environment and real query can tested to catch issues and bugs 

 
{% highlight java %}
/**
 * Test with updateSyncTimeTaken to check the update part works as expected.
 *
 * @implNote This tests adds concurrency test to check the Db access works in concurrent env.
 */
@Test
public void testUpdateSyncTime() throws Exception {
  final var updatedLibs = IntStream.range(0, 10).mapToObj((x) -> supplyAsync(() -> {
    final var library = createGenericLibrary(MAVEN);
    final var savedLib = libraryRepo.save(library);
    libraryModifyService.updateSyncTimeTaken(savedLib, 1000L);
    return libraryLookupService.getById(savedLib.getId());
  })).map(CompletableFuture::join).collect(toList());
  updatedLibs.forEach(library -> assertEquals(1000L, library.getSyncTimeTaken()));

{% endhighlight %}
 

So, the above tests the concurrency aspect of the updates and uses data JPA test to do real updates and test would have no meaning if the repo interface were mocked. 

 

Testing reflective code always has challenges as there is tendency to mock interfaces which me executed only at runtime. So traditional code coverage metrics may not tell if the code is well tested.

"Coverage is cheap, Can it catch bugs? is the real deal".

So when you mock everything u test nothing. One may get good code coverage but cannot catch bugs

"Coverage is cheap, Can it catch bugs? is the real deal".

