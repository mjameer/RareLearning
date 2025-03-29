
https://pdconfluence.tp-link.com/display/cloudshare/Tech+Design+-+Police+Response+Support+-+Azure+Integration
I'm reviewing MJ's tech design. It has some great content! I'm going to be very detailed here so as a team we can align on expectations and what's actually beneficial and to whom. My comments and thoughts below are meant to be conversational and thought provoking, NOT me micromanaging tech design content.
 
Tech designs should be done before implementation and going forward that will help clarify what type of content to put in these documents. For example, when I see class methods noted, it feels like superfluous information, unless there are comments as to why they are particularly important.
 
The requirements section is missing. A summary and/or a link to a requirements doc is great.

The Table of Contents is missing. It's a Confluence macro that can be inserted.
 
Since this is a new region, it would be helpful to understand deployment model and expectations. For example, understanding that TSP directly connects to S3 and Azure instead of via IoT/Tapo is helpful. Perhaps a deployment diagram should be added to https://pdconfluence.tp-link.com/display/cloudshare/Onboarding+-+TSP with an updated view demonstrating that TSP is only deployed in APS and it connects directly to storage providers around the world. This feels important from a performance perspective and overall architecture understanding.
All that to say, as we create tech design docs for new projects, let's think about what updates should be made to the top-level onboarding details or general architecture diagrams. Imagine a new person joined and what there experience would be if the onboarding doc didn't make sense unless the also read through 23 projects worth of tech designs







