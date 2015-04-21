---
title:	"Noobs, This is How We Test APIs"
date:	2015-04-22
---
(Disclaimer: I am a noob myself) Should you follow Test Driven Development (TDD) when you are just a beginner? Well that's exactly the right time to start, because it is always easier to learn than unlearn. I came across this choice while building a REST API backend for a project I am a part of. Here is my research.

List of Options: [QUnit](https://qunitjs.com/), [Unitjs](http://unitjs.com/), [Mocha](http://mochajs.org/), [Chakram](https://github.com/dareid/chakram), [Frisbyjs](http://frisbyjs.com/) et cetera

Just considering Chakram and Frisbyjs because all others are testing frameworks in a more general sense, for testing everything in your javascript app. We just need to test our API endpoints, so we don't need all the extra code.

Chakram is built on Mocha and [Chai](http://chaijs.com/), and here is a sample code for I don't know what:

{% highlight javascript %}
it("should expose any errors in the chakram response object", function () {
 return chakram.get("not-valid")
 .then(function(obj) {
     expect(obj.error).to.exist.and.to.be.a("object");
 }); 
{% endhighlight %}

Notice the amount of syntax that we will have to learn in order to write a test which doesn't even make sense (to noobs) in first glance. Negative points to Chakram. And no way to test Multipart uploads. 'Nuff said. (Check [this](https://github.com/dareid/chakram/blob/master/test/assertions/json.js) full API tests)

On the other hand, Frisbyjs' code sample:

{% highlight javascript %}
frisby.create('Ensure we are dealing with a teapot')
  .get('http://httpbin.org/status/418')
    .expectStatus(418)
.toss()
{% endhighlight %}

Simple, nice and intuitive! Also supports [Multipart uploads](https://github.com/vlucas/frisby/blob/master/examples/httpbin_multipart_spec.js) and [file streams](https://github.com/vlucas/frisby/blob/master/examples/httpbin_binary_post_put_spec.js).

I rest my case.




