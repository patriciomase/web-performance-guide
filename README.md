# Caching/Performance challenge
A guide to detect and fix performance issues in your web application, by **Patricio Maseda**, this document is based on insights gathered from years of experiencing and investigating issues in productive apps. I’ll highlight common issues and share the strategies and tools that have consistently helped me to level up application performance.

## Hypothetical scenario:

A client has complained that their site is too slow. In particular, they have identified three particular pages where the page load is slow. Armed with this knowledge, you set out to try to understand what the issue is.


### Proposed steps to debug and understand the problem:

 1. Communicate with the client: Use the regular communication channels to ask the most detailed explanation we can get about the observed issue. 
 2. Navigate the production site (if productive, if not whatever env the client is seeing) trying to reproduce. After finding the issue happening live it'd be beneficial to try to reproduce it locally if possible.
 3. Now having a better understanding we should be able to scope and diagnose (making use of the browser's network tab mainly) to know if the issue is mainly located in frontend or backend.

### Hypothetical case 1: Issues are in the frontend side:
 After taking a look to the affected pages the following issues have been observed:
 
 #### Symptom: Images take too long to show up.  
 - **Possible cause: Served assets size.** Images (and could be videos, fonts, etc.). They  are simply too big. This issue likely occurs when non-technical users upload content to cms platforms or other ways to inject content in automated ways. The browser scales down the image to show it in the right size defined by design, but the file transfer takes extra time which impacts the user experience.
 - **How to confirm the issue:** Inspect the elements in the browser. Compare the rendered size vs the original size. Loot at the element in the network tab, take a look at the file weight.
 - **Ways to fix:**
	 - manually reduce the size of the assets (ugh).
	 - Resize uploaded content in the backend before storing it. (Or make a copy with the right dimensions keeping the original one) This do not fix for existent content but will fix the issue for future elements. We could run a script to execute the same process for the historical data we already have. We can also consider to create several sizes at this stage so we can cover different devices in a better way and also have other sizes available like for example for thumbnails.
	 - Another alternative approach would be to resize at request time. Use query params to query for an specific size. We'd need to have a service in place to do the transformation on the fly, cache the resized asset and return. Third party services like `Cloudinary` work in this way and could be a relatively easy workaround to implement without commiting to an excesive development effort.
 - **Additional improvements:**
	 - Check the file format (mostly for images). Try to use webp.

---

#### Symptom: Page remains blank or showing minimal content for a noticeable time
- **Possible cause: Blocking scripts.** Some third-party scripts (it could be analytics or ads) are blocking the browser's main thread before the actual content even shows up. The browser is stuck waiting while these scripts load, which generates unnecessary delays.
- **How do we confirm:** 
	- Take a look at the code. Look for `<script/>` tags not including the `defer`  or `async` attribute. 
	- Go to the network tab and take a look to the trace. 
	- Run lighthouse and take a look to the diagnostics.
- **How to fix:**
	- Add `defer` or `async` to the script tag in the html.
	- Consider loading the script dynamically (Inject the `<script` tag with js) when it is needed.
   	- Consider using a third party service to load scripts like GA (Google Analytics).

---

#### Symptom: Page feels slow and unresponsive
- **Possible cause: Rendering issues, unnecessary processing.** If the page feels unresponsive after loading or after executing certain actions it could be the case we entered in a rerender loop. Another good indicator of this could be elements slightly flashing in the webpage.
- **How do we confirm**
	- Use the React Profiler (if using React) to identify components that are re-rendering too often.
    	- Check if effects or event listeners are firing frequently when they don’t need to.
     	- Reproduce the issue locally and add logs trying to identify the place where the unwanted rerenders are fired.
- **How to fix**:
    	- Optimize component rendering by using memoization, look what effects are being chained and break the loop.
    	- debounce expensive tasks so they run less often. (Things like executing events when mouse moves or keyboard typing)
    	- Remove or refactor unnecessary effects and ensure that data-fetching or resource-intensive tasks happen only when needed.

---

### Hypothetical case 2: Issues are in the backend side:
#### Symptom: The whole page or a block of content is noticeable slow to load or after sending a form with data it takes long time to show some response to the user.
In the browser's network tab we can see how some XHR requests with data needed to draw the page content are taking more than expected. 
- **Possible cause: database query underperforming** Too many joins, too much historical data, lack of indexes or db instances struggling to write and read at the same time could be the most common causes.
- **How do we confirm:** The easiest way is to have a third party monitoring tool like `New Relic` in place. Where we can see slow queries in an APM dashboard. The hard way would be to turn on database query log (or manually log suspicious queries in the app), then manually run those queries in a db client like `PGAdmin` to confirm response time.
- **What do we do to fix:** We would fix this issue in different ways depending on the issue itself.
	- If the problematic query can be optimized just do it. Try to avoid too many joins or subqueries. Analize the possibility of splitting the query and do a second iteration of simpler queries (for example bringing resources by id) to get the missing data.
	- If we see lot of filtering in the query maybe it is time to analyze and propose to have a different service like `elasticsearch` in place to do fulfill that role. Elasticsearch is much more efficient to handle large sets of data and complex filters, freeing up our main database from heavy search queries.
	- If the slowliness is not constant, it appears at certain time during the day check if it matches with user traffic spikes or with times when we run automatic processes that stress the db like a data ingestion. If the DB instances are struggling at those moments consider to scale them up, split the work in lighter chunks or schedule stress processes during low traffic volume hours.
	- If showing slightly outdated data is not an issue we can consider to add an in-memory cache layer like `redis` taking care of setting a TTL that makes sense for each case. This should reduce the number of times the app needs to query the db drastically.
---
 #### Symptom: Pages related with one specific resource (example: products) feel slow to load.
 All the site feels fluent but when accessing a product page there's a noticeable performance downgrade.
 - **Possible cause: Service used by the backend causing a delay** 
	 - In this example we mentioned pages related to products, where usually we'd have service in charge of stock and order management. If the mentioned service is slow to respond we could translate that extra time to the final user.
 - **How do we confirm:**
	 - Tools like `New Relic` or `Sentry` can give us insights about third party services response times and detect degradations. If they are not available we could try a more laborous in-house solution by logging a timestamp before and after the call to the service, including a transaction id to be able to identify and measure each transaction separately.
 - **How do we fix it:**
	 - There is not an easy way to deal with issues like this as we depend on another service. First try would be to speed up the service response if possible. If we have access to it we can analyze their logs, check for indexing or caching opportunities, or coordinate with their team to optimize their API endpoints. If that’s not an option we could implement a caching layer on our end for the data we need, reducing the amount of requests or masking the delay for the user. If caching is not an option, because we are talking about sensitive data as the available stock of an item we can decouple this logic from the original response. In this way we provide the frontend with the rest of the data needed to draw the page, and we return a separate response with this data which is prone to suffer performance degradations.

---

### Hypothetical case 3: Issues that affect both frontend and backend:
#### Symptom: All page assets seem to be slow to load. The issue is more noticeable when accessing the page from far away locations.
 -  **Possible cause: Lack of caching and CDN usage:** Assets like JS, CSS, and fonts are being delivered without any proper caching or CDN. This issue likely occurs because no caching rules were set or no CDN is present at all, the browser is downloading assets from server on every visit which increases load times and puts extra stress on the server. No cloud storage being used to store assets on each build.
 - **How do we confirm:**
	 - No cache header is present in responses.
	 - The issue is more noticeable when opening the site by first time, after clearing browser's cache or after opening an incognito window.
	 - By taking a look at project's documented diagrams about site's architecture. (Assuming they exist).
 - **What do we do to fix:**
	 - Locate the application build scripts. Look at where the compiled assets are being stored after a successful build happens. If they remain on server we should try to move away from that approach and store them in a cloud object storage services like AWS S3. Apart from that we should add a CDN layer in front of the content in order to cache and distribute globally to avoid underperforming in far away regions.

---
#### Symptom: The app takes long to have some meaningful content in the screen (poor first contentful paint time)
 -  **Possible cause: SPA rendering html by javascript in the browser:** This example suggests the client only highlighted 3 pages where the slowliness was noticeable. This issue affects the system as a whole but it could be the case some pages are more affected than others because of the nature of their content. When we do client side rendering, JS chunks are being delivered to the browser on page load among with an almost empty html file where these JS chunks get imported. All the html page content is being generated by the browser when js execution phase begins. This generates unnecessary delays and loading spinners being shown (In the best cases). Backend is also impacted receiving more traffic in the form of XHR requests to fill the page content usually for each page section. This could be solved by SSRing the whole page and returning just one response including content that we'd like to show immediately.
 - **How do we confirm:**
	 - Look at the initial request in the browser's network tab. Check the body of that request. Does it contain the actual page content in there? If it just contains a blank div with a class and a bunch of scripts you're fully rendering in client side which is slow and impractical and affects in a heavier way to clients with less resources available like phones and tablets.
 - **How do we fix the issue:** Unfortunately there is not an easy way to solve it as it goes very deep in how the system has been thought from the beginning. This scenario is usually being found in apps developed in the first react/angular years when SPAs where fashion. To properly fix this a migration to a server side render technology is required. (Something like `Nextjs`). In a perfect world we should be able to reuse the components we already have working but now we should be able to render in server.
 - **Key benefits:**
	 -  **Faster initial load:** Instead of starting with an empty html file and waiting for JS to build out the page, the server returns a fully formed html. This means users actually see content right away rather than staring at blank screens or spinners. Caching is also easier as less requests are needed to fully build the page.
    
	-   **Improved perceived performance:** Since the content is already there, even slow connections or less capable devices have something meaningful to show almost immediately. Users won’t feel like the site is sluggish or stuck.
    
	-   **More robust user experience:** If something goes wrong with the client-side JS (like a script failing), the user still sees an initial view of the page. You’re not leaving them with a broken layout or blank screen.

- **Additional benefits:**
	-   **Better SEO:** Search engines do not like an empty html and a pile of JS chunks. This change helps pages rank higher, since it’s easier for them to understand site’s structure and content. This also applies for any other crawler like the ones from social networks.

--- 
