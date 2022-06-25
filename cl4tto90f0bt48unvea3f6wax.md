## A deliverable for designing a system for a video streaming platform startup

*The motivation for this post was to create a template to compose deliverables of an architecture approach that is presentable to a potential client. We wanted this template to simple enough to be picked up by any of our engineers and serve as a first step into engineering-focused consulting prior to creating more specialized documents of software architecture.*

*Original note and article: https://polished-voyage-ff0.notion.site/A-deliverable-for-designing-a-system-for-a-video-streaming-platform-startup-ed88485585fc46ba8f2f3ad97a65b376*

Ever since we’ve launched [Dwarves Discord](https://discord.gg/dwarvesv), we’ve been able to network with all sorts of people and share knowledge with a growing community that loves tech and blockchain just like us. We’ve created study groups for topics that explores the newest tech as well as to cover our basics as engineers. In our software modeling study group, we’ve come up with a basic structure followed up with an architecture kata that is straightforward to understand.

## On the video streaming platform architecture

For our kata, we’ve chosen to do a video streaming platform directed for a startup company; more specifically, a video streaming platform that’s very similar to YouTube in that we allow any users to be curators and viewers of video content on the site. Just like in consulting, there is a lot of prep work and research that happens behind the scenes before creating an architecture presentation as the proposal for the client. For this kata, we post our research questions and notes to a [Workflowy bullet](https://www.workflowy.com/s/design-a-video-strea/DbqxIVsOgp2Ab2rd). The approach is a bit impromptu and intentionally sparse, but the outlines in the Workflowy represents some of our practices of the techniques we use in the study group as well our train of thoughts on researching the topic.

The video streaming platform itself has relatively a few core basic concepts and requirements:

- Allow users to upload video content
- Allow users to search other users’ video content

To ground our kata to reality, there are a few items that are of high concern to the client inquiring the architecture. As such, our most relevant objectives should consist of:

- Finding methods of creating revenue on this platform
- Finding a way to model the platform in a way that is cost-effective
- Finding an architecture that is appropriate for a startup

## Deciding on the architecture

If we were in the shoes of our client, we would definitely have some reservations on building our architecture from scratch. Luckily a video streaming platform isn’t completely a foreign concept and we can look up examples of existing platforms to answer our concerns. We can take examples from Netflix, YouTube, and others.

For Netflix, they use a microservices architecture for their platform. Although the type of platform is different than what we’re looking for in our kata, it is a perfectly valid architecture approach for our practice. Just on notability, their story from a monolithic application to small, loosely coupled microservices along with open-source patterns with [video-encoding](https://netflixtechblog.com/high-quality-video-encoding-at-scale-d159db052746) were novel and industry leading enough to win them the [Jury Innovation Awards in 2015](https://jaxenter.com/jax-awards-2015). However, such an architecture is absolutely overkill for a startup.

For YouTube, their full engineering story is not as obvious, but we have a glimpse to their initial story and their direction. At first, the YouTube stack back in 2008 was [a simple LAMP stack](http://highscalability.com/youtube-architecture) with a python library for optimization and a separate web server for video. Later down the road, it became much less of a LAMP stack and more of a Content Delivery Network (CDN) that introduced a lot more [data-centric optimizations to solve scalability issues](http://highscalability.com/blog/2012/3/26/7-years-of-youtube-scalability-lessons-in-30-minutes.html). Although an interesting story to pursue, we aren’t here to design a data center for our client.

At the end of the day, these platforms focused on content delivery optimizations to solve scalability issues rather than changing their whole system architecture - which means our architecture may be **secondary** to the **necessity of a CDN.** Notable similarities across all of these existing platforms is that use an external service or establish their own CDN in addition to having a dedicated service to transcode videos into multiple formats, despite their different system architectures. For our case, we can definitely offload video distribution work to a CDN and let them handle it. This leaves us with only a few remaining concerns:

- Where do we offload the work for transcoding videos?
- What architecture is cost effective enough and is easy to evolve?

One optimal architecture to fit our requirements would be a service-based architecture. It sits in between a monolith and a microservices architecture in a way that’s not absolutely overkill for the engineers that will have to get involved. It’s an architecture that is easy to reason about and fit domain patterns that allows us to present it to the client without immediately going into technical details. The system architecture also meets our objectives to create an architecture that is appropriate for the startup and is easy to evolve.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157336005/pbAhe9hxJ.png align="left")
*Figure 1. Basic topology of the service-based architecture style from “Fundamentals of Software Architecture: An Engineering Approach 1st Edition”*

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157418394/U7KDyLzKR.png align="left")
*Figure 2. Service-based architecture characteristics ratings from “Fundamentals of Software Architecture: An Engineering Approach 1st Edition”*

## Diagramming the architecture

We use the [C4 model](https://c4model.com/) to diagram domain abstractions of our video streaming kata. These diagrams are particularly useful for us and for clients, as they represent system architectures in a semantically consistent format that can stand by itself with little to no prior context. The abstractions of the diagram go down 4 levels:

1. **System Context diagram** - shows the context of the entire system with related entities
2. **Container diagram** - a high level shape of the architecture and how it fits the IT environment
3. **Component diagram** - decompose containers into components to show implementation abstractions
4. **Code** - shows how the component is implemented as programmable code

For our kata, we won’t show the fourth level of the C4 model, as we don’t have overly complex components that motivates the need to elaborate it in code. We do have examples of domain-centric diagrams which gives technical staff and developers general guidelines on how requirements should be implemented.

### Context diagram

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157449537/RRlvGDd2o.png align="left")

### Container diagram

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157501762/xKMP-e9PR.png align="left")

### Component diagrams

#### Account service

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157558030/EHzqWMcaf.png align="left")

#### Content service

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157572754/JurtcSRBG.png align="left")

#### Analytics service

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157594139/buRRCmWWn.png align="left")

#### Upload service

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157599104/yuhJXDwOk.png align="left")

#### Transcoder service

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157628209/aq4SGDUdI.png align="left")
*Figure 3. Netflix system design - Transcoding a video https://medium.com/@Aditya_Gupta_DS/netflix-system-design-29ab4b1f2fb1*

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157648975/3FKgJWegG.png align="left")

---

## Domain-centric diagrams for developers

These diagrams can consist of state machines, UML diagrams, entity relationship diagrams, etc. For this kata, we use BPMN diagrams (think of it as a kind of state machine with details) and sequence diagrams to explain request response flows of notable services. Realistically, if we are coordinating closely with a client and the developers we expect to be in the project, there would be many more developer diagrams for implementations that touch on specific details of requirements.

### Video upload

#### BPMN diagram

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157687986/fOukF4Dzb.png align="left")

#### Sequence diagram

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157693159/yjmBQ_tr_.png align="left")

### Content search

#### BPMN diagram

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157726663/0ZdB2tNAF.png align="left")

#### Sequence diagram

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656157731682/o2plI4ncp.png align="left")

## Closing thoughts

We were able to create an approach style and template we thought best fit our concerns of a kind of scaffold to lean onto and a good starting point in composing software architectures for real world clients. Our upcoming katas will explore other types of software architecture that will most likely include different starting diagrams or include more technical notes. Overall, our general approach in creating the deliverable will not likely change, and that is just really exciting for us. This is a first of many milestones we’ve completed for our software modeling study group, and hopefully we can introduce a similar sense of simplicity for much more complex patterns for our upcoming architecture practices.

This would not have been possible without the novel inputs and initial research provided by our colleagues. We’ve definitely learned some very vital life lessons along the way. If you feel that you are in a situation where you can’t progress, ask for help. Maybe you just need a bit of small input or a push from a colleague to move your ideas forward, don’t hesitate to reach out to your colleagues. We find the smallest steps end up being the most important ones.