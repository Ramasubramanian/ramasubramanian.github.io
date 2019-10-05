## Why Me? Or my industry?

**TLDR;** Only IT industry has zillion ways to do same thing like writing REST APIs building UIs and storing data. Other industries e.g. carpentry, automobile seem to have some form of standard with less deviations and feel more consistent across the world. Too many ways to do same thing leads to chaos and confusion. Or may be I am wrong and it is human nature irrespective of industry to complicate lives. 

**If you have more time:**

Being with the software industry for past 15 years I have been thinking about how our industry is different from others
or why we have so many ways to do things while others have lesser number of standardised processes, techniques etc. This thought has been flashing once in a while for the past 8 years. When i saw a tweet from Jaana B Dogan.
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Programming is mostly about transforming from one data format to another. Let&#39;s stop pretending it is something else.</p>&mdash; Jaana B. Dogan (@rakyll) <a href="https://twitter.com/rakyll/status/1178723806128549889?ref_src=twsrc%5Etfw">September 30, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

This thought again flashed over my mind and I ranted again with below tweet.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Have been feeling it for almost 8 years now. Just zillions of frameworks, languages and methodologies to do This constitutes 90%+ of our industry. User -&gt; transform -&gt; storage and storage -&gt; transform -&gt; user <a href="https://t.co/mgM7oQyabP">https://t.co/mgM7oQyabP</a></p>&mdash; ramasubramanian (@ramsankar83) <a href="https://twitter.com/ramsankar83/status/1178837218816741376?ref_src=twsrc%5Etfw">October 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Most of the work we do in our IT industry can be summarized below:
1. (a) User [Interface] -> (b) Transform -> (c) Storage 
2. (a') Storage -> (b') Transform -> (c') User [Interface]

> Note: I am aware of some parts of the industry not doing this stuff and really working on tricky problems like system software and large scale distributed computing stuff I am not taking that side into account as it is very less compared to what we are talking here.

The complexity of the transformation logic varies from system to system. Some are simple CRUD systems (looked down upon) some do processing and integration with umpteen other systems and considered good to work with. 

If we think about ways to do this and variations I can summarize them quickly below:

a, a' => HTML, CSS, Bootstrap, Flexbox, JS, AngularJS, VueJS, ReactJS, *JS, ASP, JSP, Server side rendering,..... (Zillion + 1)

b, b' => REST API frameworks, Spring boot/MVC, Python (Django....), Dropwizard, Rust, Haskell, NodeJS, route based, MVC, Lift,..... (Zillion + 1)

c, c' => RDBMS, NoSQL, Streaming, Postgres, Oracle, Big Data, ....... (Zillion + 1)

a and a' are user facing components, with current modernisation and digital world with stiff competition users will feel bored and move away with redundant or monotonous UI so need for fanciness and maximum convenience is a mandate especially for B2C systems so it is understandable why UX (CSS) has so many variants optimised for different kind of UI requirements. 

I am tempted to compare this with a carpenter creating a wooden sofa set or dining table. Or a chemical engineer creating some compound. In the case of a carpenter their users might ask for different designs in the edges, art works, varied thickness, colours etc. So multiple ways of achieving what user wants is justified.

b and b' are all similar across systems in the sense there needs to be a remote machine running somewhere listening to requests which are typically HTTP. Once a request comes receive the same and either send new transformed data from storage as response or save the incoming data after transformation into storage. The scale may vary, the complexity of the transformation may vary, dependencies like external systems may vary but the backbone is the same. Same is applicable to c and c' as well.

Comparing and contrasting this to a carpenter sofa creation process it is cutting the raw wood into shapes which are sub-components of the sofa and assembling i.e. joining them using nails. Though the wood material and polishing process, finishing touches etc might vary based on what a user asked the core design like frame, legs, back rest etc and the creation process might be more or less similar across carpenters all over the world, may be there are some variations like the [Japanese techniques](https://www.demilked.com/ancient-japanese-carpentry-techniques-kobayashi-kenkou/) of joining stuff without nails but definitely not in zillions. The sawing, smoothening process might be same almost all over the world. 

Similarly if we take a car manufacturer the process to create chasis might be same across the world with less number of variations. But why does IT industry have so many ways to do one thing? 

a. may be we are wrong here since car chasis or sofa design once designed/created never changes but the REST APIs we develop keep changing over time due to business requirements. Even if there are changes it does not warrant so many variations to achieve the same result. 

b. May be the systems we build do not exist in real world but aid real world so we are insulated from the laws/horros of real physical world for e.g. once a sofa base is created it can changed only so much but given enough time we can change what we have built with software completely? I mean the cost of re-engineering/engineering is way high in other industries but not so high in our industry? especially when you can make people work 12 - 14 hours 6 days a week? 

This point was the result of a follow up discussion with a good friend of mine and a senior industry person [@sundararajan_a](https://twitter.com/sundararajan_a). 

If you get the design of a chasis wrong then you cannot go back to the chasis manufaturer who has setup a high cost industrial process to manufacture millions of such chasises to change it considerably. May be that is why they are more prudent and variety averse unlike us who take things for granted and rewrite or redesign often? Is it an advantage or a disadvantage? 

May be we should think of it as "see we have the freedom to rewrite whole systems unlike other industries which is why we are awesome?"

c. If we don't try new stuff we can never improve or innovate so there should be alternatives to mainstream processes of course but how much is too much? For example there wouldn't be a Tesla if we thought why deviate from standard process of manufacturing petrol/diesel engines. So variety is required but are we as an industry providing too much variety to the developers which leads to confusion and chaos? 

d. Just because we can should we devise so many ways and tools to build systems? Is consistency so bad or a far fetched thing to ask for in IT industry?

Adding fuel to already burning fire we have the complex Cloud platforms like AWS, GCP, Azure etc which further complicate the scenario by themselves offering multiple ways to do the same thing and god knows how to decide the right choice. 

Organisations like IETF, W3C etc are trying to standardise stuff by providing specifications, on UI side initiatives like Material UI etc are trying to bring some consolidation but still I feel we have zillion ways to implement and run those specs and it is not helping. 

Having said all this may be I am wrong, I am just thinking "the grass is greener on the other side" if we ask an automobile engineer or a carpenter they will have their share of complicated stuff in their respective industries from their point of view, I don't know to be honest. 

But I still feel we have too many tools/frameworks/processes at our disposal which confuse us and complicate lives rather than simplify/aid us and should tone down the varieties available. Thoughts?