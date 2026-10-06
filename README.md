# Doxbin.com Endpoint Mapper for the Chrome DevTools Console

# Usage
<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/65edfbde-4cc1-47ec-9c40-5966578d6fc9" />
<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/0a44c4b1-5959-48f7-8d48-89b03bb4463b" />

1. go to doxbin.com
2. press f12 or right click and inspect element then go to console
3. next in the console type allowpasting then you will paste the following code

```js
(async () => {
    "use strict";

    /*
     * ============================================================
     *  UNIVERSAL CLIENT-SIDE ENDPOINT MAPPER
     *  Passive/static + live browser observation
     * ============================================================
     */

    const TARGET_ORIGIN = location.origin;
    const TARGET_HOST = location.hostname;

    const endpoints = new Map();
    const scriptsSeen = new Set();

    const normalizeMethod = method => {
        if (!method) return "UNKNOWN";

        method = String(method).toUpperCase().trim();

        return [
            "GET",
            "POST",
            "PUT",
            "PATCH",
            "DELETE",
            "HEAD",
            "OPTIONS"
        ].includes(method)
            ? method
            : "UNKNOWN";
    };

    const classify = path => {
        if (/\/graphql(?:\/|$)/i.test(path))
            return "GraphQL";

        if (/\/(?:api|api\/v\d+)(?:\/|$)/i.test(path))
            return "REST/API";

        if (/\/(?:rpc|jsonrpc)(?:\/|$)/i.test(path))
            return "RPC";

        if (/(?:ajax|async|xhr)(?:\/|$)/i.test(path))
            return "AJAX";

        if (/\/(?:oauth|auth|login|logout|session|token)(?:\/|$)/i.test(path))
            return "Authentication";

        if (/\/(?:upload|download|file|files|media)(?:\/|$)/i.test(path))
            return "File";

        if (/\.(?:json|xml|yaml|yml)$/i.test(path))
            return "Data";

        return "Page/Resource";
    };

    const addEndpoint = (
        rawUrl,
        method = "UNKNOWN",
        source = "unknown",
        metadata = {}
    ) => {
        if (!rawUrl) return null;

        let url;

        try {
            url = new URL(String(rawUrl), location.href);
        } catch {
            return null;
        }

        /*
         * Strict same-origin inventory.
         */
        if (url.origin !== TARGET_ORIGIN)
            return null;

        /*
         * Remove fragments because they are client-side only.
         */
        url.hash = "";

        const normalizedMethod = normalizeMethod(method);

        /*
         * Route key deliberately excludes query values.
         * /api/foo?id=1 and /api/foo?id=2 become one endpoint.
         */
        const key =
            `${normalizedMethod} ${url.origin}${url.pathname}`;

        let entry = endpoints.get(key);

        if (!entry) {
            entry = {
                method: normalizedMethod,
                url: `${url.origin}${url.pathname}`,
                path: url.pathname,
                queryPatterns: new Set(),
                sources: new Set(),
                classifications: new Set(),
                parameters: new Set(),
                bodies: new Set(),
                headers: new Set(),
                count: 0
            };

            endpoints.set(key, entry);
        }

        entry.count++;

        if (url.search)
            entry.queryPatterns.add(url.search);

        entry.sources.add(source);
        entry.classifications.add(classify(url.pathname));

        if (metadata.parameters) {
            for (const p of metadata.parameters) {
                entry.parameters.add(String(p));
            }
        }

        if (metadata.body) {
            entry.bodies.add(
                typeof metadata.body === "string"
                    ? metadata.body
                    : JSON.stringify(metadata.body)
            );
        }

        if (metadata.headers) {
            for (const h of metadata.headers) {
                entry.headers.add(String(h));
            }
        }

        return entry;
    };

    const extractParameters = url => {
        try {
            const u = new URL(url, location.href);
            return [...u.searchParams.keys()];
        } catch {
            return [];
        }
    };

    const addFromCode = (
        code,
        source,
        scriptUrl = null
    ) => {
        if (!code) return;

        /*
         * --------------------------------------------------------
         * Absolute same-origin URLs
         * --------------------------------------------------------
         */

        const absoluteURL =
            /["'`](https?:\/\/[^"'`\s<>]+)["'`]/gi;

        let match;

        while ((match = absoluteURL.exec(code))) {
            addEndpoint(
                match[1],
                "UNKNOWN",
                `${source}:absolute-url`
            );
        }

        /*
         * --------------------------------------------------------
         * Root-relative URLs
         * --------------------------------------------------------
         */

        const rootURL =
            /["'`](\/[^"'`\s<>]{1,1000})["'`]/g;

        while ((match = rootURL.exec(code))) {
            const value = match[1];

            /*
             * Avoid treating ordinary fragments/assets as APIs,
             * while still retaining everything that resembles
             * a route.
             */
            if (
                value.startsWith("//") ||
                value.startsWith("/static/") ||
                /\.(?:css|jpg|jpeg|png|gif|svg|ico|woff2?|ttf)$/i.test(value)
            ) {
                continue;
            }

            addEndpoint(
                value,
                "UNKNOWN",
                `${source}:root-url`,
                {
                    parameters: extractParameters(value)
                }
            );
        }

        /*
         * --------------------------------------------------------
         * fetch("/route", {method:"POST"})
         * --------------------------------------------------------
         */

        const fetchRegex =
            /fetch\s*\(\s*["'`]([^"'`]+)["'`]\s*(?:,\s*\{([\s\S]*?)\})?\s*\)/gi;

        while ((match = fetchRegex.exec(code))) {
            const url = match[1];
            const options = match[2] || "";

            const methodMatch =
                options.match(
                    /\bmethod\s*:\s*["'`](GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS)["'`]/i
                );

            const bodyMatch =
                options.match(
                    /\bbody\s*:\s*([^,\n}]+)/i
                );

            const headersMatch =
                options.match(
                    /\bheaders\s*:\s*(\{[\s\S]*?\})/i
                );

            addEndpoint(
                url,
                methodMatch
                    ? methodMatch[1]
                    : "UNKNOWN",
                `${source}:fetch`,
                {
                    parameters: extractParameters(url),
                    body: bodyMatch
                        ? bodyMatch[1]
                        : null,
                    headers: headersMatch
                        ? [headersMatch[1]]
                        : []
                }
            );
        }

        /*
         * --------------------------------------------------------
         * axios.get/post/put/patch/delete(...)
         * --------------------------------------------------------
         */

        const axiosRegex =
            /axios\.(get|post|put|patch|delete|head|options)\s*\(\s*["'`]([^"'`]+)["'`]/gi;

        while ((match = axiosRegex.exec(code))) {
            addEndpoint(
                match[2],
                match[1],
                `${source}:axios`,
                {
                    parameters: extractParameters(match[2])
                }
            );
        }

        /*
         * --------------------------------------------------------
         * XMLHttpRequest
         *
         * xhr.open("POST", "/api/test")
         * --------------------------------------------------------
         */

        const xhrRegex =
            /\.open\s*\(\s*["'`](GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS)["'`]\s*,\s*["'`]([^"'`]+)["'`]/gi;

        while ((match = xhrRegex.exec(code))) {
            addEndpoint(
                match[2],
                match[1],
                `${source}:xhr`,
                {
                    parameters: extractParameters(match[2])
                }
            );
        }

        /*
         * --------------------------------------------------------
         * navigator.sendBeacon(...)
         * --------------------------------------------------------
         */

        const beaconRegex =
            /navigator\.sendBeacon\s*\(\s*["'`]([^"'`]+)["'`]/gi;

        while ((match = beaconRegex.exec(code))) {
            addEndpoint(
                match[1],
                "POST",
                `${source}:beacon`
            );
        }

        /*
         * --------------------------------------------------------
         * EventSource
         * --------------------------------------------------------
         */

        const eventSourceRegex =
            /new\s+EventSource\s*\(\s*["'`]([^"'`]+)["'`]/gi;

        while ((match = eventSourceRegex.exec(code))) {
            addEndpoint(
                match[1],
                "GET",
                `${source}:eventsource`
            );
        }

        /*
         * --------------------------------------------------------
         * WebSocket
         * --------------------------------------------------------
         */

        const websocketRegex =
            /new\s+WebSocket\s*\(\s*["'`]([^"'`]+)["'`]/gi;

        while ((match = websocketRegex.exec(code))) {
            try {
                const ws = new URL(
                    match[1],
                    location.href
                );

                if (
                    ws.protocol === "ws:" ||
                    ws.protocol === "wss:"
                ) {
                    addEndpoint(
                        ws.href,
                        "WS",
                        `${source}:websocket`
                    );
                }
            } catch {}
        }

        /*
         * --------------------------------------------------------
         * Common endpoint variables
         * --------------------------------------------------------
         */

        const endpointAssignment =
            /(?:url|endpoint|apiUrl|apiURL|route|path)\s*:\s*["'`]([^"'`]+)["'`]/gi;

        while ((match = endpointAssignment.exec(code))) {
            addEndpoint(
                match[1],
                "UNKNOWN",
                `${source}:endpoint-variable`
            );
        }

        /*
         * --------------------------------------------------------
         * location.href / location.assign / location.replace
         * --------------------------------------------------------
         */

        const navigationRegex =
            /(?:location\.(?:assign|replace)|window\.location\s*=)\s*["'`]([^"'`]+)["'`]/gi;

        while ((match = navigationRegex.exec(code))) {
            addEndpoint(
                match[1],
                "GET",
                `${source}:navigation`
            );
        }

        /*
         * --------------------------------------------------------
         * URLSearchParams / query-looking endpoint strings
         * --------------------------------------------------------
         */

        const apiLike =
            /["'`](\/(?:api|api\/v\d+|graphql|rest|rpc|ajax|oauth|auth|session|token|upload|download)(?:\/[^"'`\s]*)?)["'`]/gi;

        while ((match = apiLike.exec(code))) {
            addEndpoint(
                match[1],
                "UNKNOWN",
                `${source}:api-pattern`
            );
        }
    };

    /*
     * ============================================================
     * BEGIN
     * ============================================================
     */

    console.clear();

    console.log(
        "%cUNIVERSAL ENDPOINT MAPPER",
        "font-size:20px;font-weight:bold"
    );

    console.log(
        "%cTarget:",
        "font-weight:bold",
        TARGET_ORIGIN
    );

    /*
     * Current page.
     */

    addEndpoint(
        location.href,
        "GET",
        "current-page"
    );

    /*
     * ============================================================
     * HTML
     * ============================================================
     */

    document
        .querySelectorAll(
            "a[href],form,iframe[src],script[src],img[src]," +
            "link[href],video[src],audio[src],source[src]," +
            "[action],[data-url],[data-endpoint],[data-api]"
        )
        .forEach(element => {

            if (element.matches("a[href]")) {
                addEndpoint(
                    element.href,
                    "GET",
                    "html:a"
                );
            }

            if (element.matches("form")) {
                const method =
                    normalizeMethod(
                        element.getAttribute("method") || "GET"
                    );

                const fields =
                    [...element.elements]
                        .filter(x => x.name)
                        .map(x => x.name);

                addEndpoint(
                    element.action || location.href,
                    method,
                    "html:form",
                    {
                        parameters: fields
                    }
                );
            }

            for (const attr of [
                "src",
                "href",
                "action",
                "data-url",
                "data-endpoint",
                "data-api"
            ]) {
                const value =
                    element.getAttribute(attr);

                if (!value) continue;

                const method =
                    attr === "action"
                        ? normalizeMethod(
                            element.getAttribute("method") || "GET"
                        )
                        : "GET";

                addEndpoint(
                    value,
                    method,
                    `html:${attr}`
                );
            }
        });

    /*
     * ============================================================
     * INLINE SCRIPTS
     * ============================================================
     */

    for (const script of document.scripts) {
        if (script.src) {
            addEndpoint(
                script.src,
                "GET",
                "script"
            );
        }

        addFromCode(
            script.textContent || "",
            "inline-script",
            script.src || null
        );
    }

    /*
     * ============================================================
     * EXTERNAL SAME-ORIGIN JAVASCRIPT
     * ============================================================
     */

    const scriptURLs = [
        ...document.scripts
    ]
        .map(x => x.src)
        .filter(Boolean)
        .filter(src => {
            try {
                return new URL(src).origin === TARGET_ORIGIN;
            } catch {
                return false;
            }
        });

    for (const src of scriptURLs) {
        if (scriptsSeen.has(src))
            continue;

        scriptsSeen.add(src);

        try {
            const response =
                await fetch(src, {
                    credentials: "include",
                    cache: "no-store"
                });

            if (!response.ok)
                continue;

            const code =
                await response.text();

            console.log(
                "%cScanning JavaScript:",
                "color:#00aaff",
                src
            );

            addFromCode(
                code,
                `external-script:${src}`,
                src
            );
        } catch (error) {
            console.warn(
                "Could not inspect script:",
                src,
                error
            );
        }
    }

    /*
     * ============================================================
     * PERFORMANCE RESOURCE INVENTORY
     * ============================================================
     */

    performance
        .getEntriesByType("resource")
        .forEach(entry => {
            addEndpoint(
                entry.name,
                "GET",
                `performance:${entry.initiatorType}`
            );
        });

    /*
     * ============================================================
     * LIVE FETCH MONITOR
     * ============================================================
     */

    if (!window.__endpointMapperInstalled) {

        const originalFetch =
            window.fetch;

        window.fetch =
            async function (...args) {

                try {
                    const request =
                        args[0];

                    const options =
                        args[1] || {};

                    if (request instanceof Request) {

                        addEndpoint(
                            request.url,
                            request.method,
                            "live:fetch",
                            {
                                headers:
                                    [...request.headers.keys()]
                            }
                        );

                    } else {

                        addEndpoint(
                            String(request),
                            options.method || "GET",
                            "live:fetch"
                        );
                    }

                } catch {}

                return originalFetch.apply(
                    this,
                    args
                );
            };

        /*
         * ========================================================
         * LIVE XHR
         * ========================================================
         */

        const originalOpen =
            XMLHttpRequest.prototype.open;

        XMLHttpRequest.prototype.open =
            function (
                method,
                url,
                ...rest
            ) {

                addEndpoint(
                    url,
                    method,
                    "live:xhr"
                );

                return originalOpen.call(
                    this,
                    method,
                    url,
                    ...rest
                );
            };

        /*
         * ========================================================
         * LIVE WEBSOCKET
         * ========================================================
         */

        const OriginalWebSocket =
            window.WebSocket;

        window.WebSocket =
            function (url, protocols) {

                addEndpoint(
                    url,
                    "WS",
                    "live:websocket"
                );

                return protocols === undefined
                    ? new OriginalWebSocket(url)
                    : new OriginalWebSocket(
                        url,
                        protocols
                    );
            };

        window.WebSocket.prototype =
            OriginalWebSocket.prototype;

        window.__endpointMapperInstalled = true;
    }

    /*
     * ============================================================
     * FORMAT OUTPUT
     * ============================================================
     */

    const serialize = () =>
        [...endpoints.values()]
            .map(e => ({
                method: e.method,
                url: e.url,
                path: e.path,

                queryPatterns:
                    [...e.queryPatterns],

                sources:
                    [...e.sources],

                classifications:
                    [...e.classifications],

                parameters:
                    [...e.parameters],

                bodies:
                    [...e.bodies],

                headers:
                    [...e.headers],

                observations:
                    e.count
            }))
            .sort((a, b) =>
                `${a.path}${a.method}`
                    .localeCompare(
                        `${b.path}${b.method}`
                    )
            );

    window.__endpointMap = {
        target: TARGET_ORIGIN,
        hostname: TARGET_HOST,
        generated:
            new Date().toISOString(),
        endpoints:
            serialize()
    };

    /*
     * ============================================================
     * API-ONLY VIEW
     * ============================================================
     */

    window.__apiMap =
        () =>
            serialize().filter(x =>
                x.classifications.some(c =>
                    [
                        "REST/API",
                        "GraphQL",
                        "RPC",
                        "AJAX",
                        "Authentication",
                        "File"
                    ].includes(c)
                )
            );

    /*
     * ============================================================
     * JSON EXPORT
     * ============================================================
     */

    window.exportEndpointMap = () => {

        const blob =
            new Blob(
                [
                    JSON.stringify(
                        window.__endpointMap,
                        null,
                        2
                    )
                ],
                {
                    type:
                        "application/json"
                }
            );

        const url =
            URL.createObjectURL(blob);

        const a =
            document.createElement("a");

        a.href = url;

        a.download =
            `${TARGET_HOST}-endpoint-map-${Date.now()}.json`;

        document.body.appendChild(a);

        a.click();

        a.remove();

        setTimeout(
            () =>
                URL.revokeObjectURL(url),
            5000
        );
    };

    /*
     * ============================================================
     * CSV EXPORT
     * ============================================================
     */

    window.exportEndpointCSV = () => {

        const rows = serialize();

        const escapeCSV = value =>
            `"${String(value ?? "")
                .replace(/"/g, '""')}"`;

        const header = [
            "method",
            "url",
            "path",
            "queryPatterns",
            "sources",
            "classifications",
            "parameters",
            "bodies",
            "headers",
            "observations"
        ];

        const csv = [
            header.map(escapeCSV).join(","),
            ...rows.map(row =>
                [
                    row.method,
                    row.url,
                    row.path,
                    row.queryPatterns.join(" | "),
                    row.sources.join(" | "),
                    row.classifications.join(" | "),
                    row.parameters.join(" | "),
                    row.bodies.join(" | "),
                    row.headers.join(" | "),
                    row.observations
                ]
                    .map(escapeCSV)
                    .join(",")
            )
        ].join("\n");

        const blob =
            new Blob(
                [csv],
                { type: "text/csv" }
            );

        const url =
            URL.createObjectURL(blob);

        const a =
            document.createElement("a");

        a.href = url;

        a.download =
            `${TARGET_HOST}-endpoint-map-${Date.now()}.csv`;

        document.body.appendChild(a);

        a.click();

        a.remove();

        setTimeout(
            () =>
                URL.revokeObjectURL(url),
            5000
        );
    };

    /*
     * ============================================================
     * DISPLAY
     * ============================================================
     */

    const rows = serialize();

    console.log(
        `%cTOTAL UNIQUE ROUTES: ${rows.length}`,
        "color:#00ff88;font-size:16px;font-weight:bold"
    );

    console.table(
        rows.map(x => ({
            Method: x.method,
            Path: x.path,
            Type: x.classifications.join(" | "),
            Sources: x.sources.join(" | "),
            Parameters: x.parameters.join(", "),
            Observations: x.observations
        }))
    );

    console.log(
        `%cAPI / service routes: ${window.__apiMap().length}`,
        "color:#ffaa00;font-weight:bold"
    );

    console.table(
        window.__apiMap().map(x => ({
            Method: x.method,
            Path: x.path,
            Type: x.classifications.join(" | "),
            Parameters: x.parameters.join(", "),
            Sources: x.sources.join(" | ")
        }))
    );

    console.log("");
    console.log(
        "%cCommands",
        "font-size:15px;font-weight:bold"
    );
    console.log(
        "window.__endpointMap.endpoints"
    );
    console.log(
        "__apiMap()"
    );
    console.log(
        "exportEndpointMap()"
    );
    console.log(
        "exportEndpointCSV()"
    );

    console.log("");
    console.log(
        "%cLive monitoring is active.",
        "color:#00aaff;font-weight:bold"
    );
    console.log(
        "Navigate through the authorized application normally to capture additional fetch/XHR/WebSocket endpoints."
    );

})();
```

4. Now you should see the results.

Mine were

```text
﻿
VM648:935 
(index)
Method
Path
Type
Sources
Parameters
Observations
0	'POST'	'/api/timezone/set'	'REST/API'	'inline-script:fetch'	''	1
1	'UNKNOWN'	'/api/timezone/set'	'REST/API'	'inline-script:root-url | inline-script:api-pattern'	''	2
2	'POST'	'/api/token'	'REST/API'	'inline-script:fetch'	''	1
3	'UNKNOWN'	'/api/token'	'REST/API'	'inline-script:root-url | inline-script:api-pattern'	''	2
4	'POST'	'/api/user/comment/create'	'REST/API'	'inline-script:fetch'	'user_id, key, message'	1
5	'UNKNOWN'	'/api/user/comment/create'	'REST/API'	'inline-script:root-url | inline-script:api-pattern'	'user_id, key, message'	2
6	'UNKNOWN'	'/api/user/comment/hide'	'REST/API'	'inline-script:root-url | inline-script:api-pattern'	''	2
7	'UNKNOWN'	'/api/user/comment/unhide'	'REST/API'	'inline-script:root-url | inline-script:api-pattern'	''	2
8	'GET'	'/'	'Page/Resource'	'html:a | html:href'	''	4
9	'GET'	'/hoa'	'Page/Resource'	'html:a | html:href'	''	2
10	'GET'	'/legal'	'Page/Resource'	'html:a | html:href'	''	2
11	'GET'	'/logout'	'Authentication'	'html:a | html:href'	''	2
12	'GET'	'/settings'	'Page/Resource'	'html:a | html:href'	''	4
13	'GET'	'/static/banner.jpg'	'Page/Resource'	'performance:css'	''	1
14	'GET'	'/static/base.css'	'Page/Resource'	'html:href | performance:link'	''	2
15	'GET'	'/static/default.jpg'	'Page/Resource'	'html:src | performance:img'	''	2
16	'GET'	'/static/inter.css'	'Page/Resource'	'html:href | performance:link'	''	2
17	'GET'	'/tos'	'Page/Resource'	'html:a | html:href'	''	6
18	'GET'	'/upgrade'	'Page/Resource'	'html:a | html:href'	''	2
19	'GET'	'/upload/cybersechaxZine'	'File'	'html:a | html:href'	''	2
20	'GET'	'/upload/DefconGooNLOG'	'File'	'html:a | html:href'	''	2
21	'GET'	'/upload/GabrielakaTrojan'	'File'	'html:a | html:href'	''	2
22	'GET'	'/upload/HACKFORUMSOWNER'	'File'	'html:a | html:href'	''	2
23	'GET'	'/upload/InfamousHaze'	'File'	'html:a | html:href'	''	2
24	'GET'	'/upload/KiwifarmsOwnerOwned'	'File'	'html:a | html:href'	''	2
25	'GET'	'/upload/KTOWNED'	'File'	'html:a | html:href'	''	2
26	'GET'	'/upload/KTOWNED'	'File'	'html:a | html:href'	''	2
27	'GET'	'/upload/NacashOwned'	'File'	'html:a | html:href'	''	2
28	'GET'	'/upload/NyanBB'	'File'	'html:a | html:href'	''	2
29	'GET'	'/upload/RyanDisney96Stenwick'	'File'	'html:a | html:href'	''	2
30	'GET'	'/upload/RyanLizardSquad'	'File'	'html:a | html:href'	''	2
31	'GET'	'/upload/RyanStenwickLOL'	'File'	'html:a | html:href'	''	2
32	'GET'	'/upload/Sa7anREAL'	'File'	'html:a | html:href'	''	2
33	'GET'	'/upload/SamGurvinderBhangu'	'File'	'html:a | html:href'	''	2
34	'GET'	'/upload/ShanePatrickLilly'	'File'	'html:a | html:href'	''	2
35	'GET'	'/upload/TheDeathofKEEMSTAR'	'File'	'html:a | html:href'	''	2
36	'GET'	'/upload/TryhackmeHackedel84ezine'	'File'	'html:a | html:href'	''	2
37	'GET'	'/upload/urharmlessOwned'	'File'	'html:a | html:href'	''	2
38	'GET'	'/upload/VileDotSHOwner'	'File'	'html:a | html:href'	''	2
39	'GET'	'/upload/WilliamJamesHollamonowned'	'File'	'html:a | html:href'	''	2
40	'GET'	'/upload/WoohTheGodakaZacharyDShelton'	'File'	'html:a | html:href'	''	2
41	'GET'	'/upload/ZacharyEvanEmswiler'	'File'	'html:a | html:href'	''	2
42	'GET'	'/upload/zeekill'	'File'	'html:a | html:href'	''	2
43	'GET'	'/upload'	'File'	'html:a | html:href'	''	2
44	'POST'	'/user/$%7Burl%7D'	'Page/Resource'	'inline-script:fetch'	'comment_id'	1
45	'GET'	'/user/nologin/followers'	'Page/Resource'	'html:a | html:href'	''	2
46	'GET'	'/user/nologin/following'	'Page/Resource'	'html:a | html:href'	''	2
47	'UNKNOWN'	'/user/nologin/'	'Page/Resource'	'inline-script:root-url'	''	1
48	'GET'	'/user/nologin'	'Page/Resource'	'current-page | html:a | html:href | performance:fetch'	''	9
49	'GET'	'/users'	'Page/Resource'	'html:a | html:href'	''	2
Array(49)
VM648:946 API / service routes 33
VM648:951 
(index)
Method
Path
Type
Parameters
Sources
0	'POST'	'/api/timezone/set'	'REST/API'	''	'inline-script:fetch'
1	'UNKNOWN'	'/api/timezone/set'	'REST/API'	'	'inline-script:root-url | inline-script:api-pattern'
2	'POST'	'/api/token'	'REST/API'	''	'inline-script:fetch'
3	'UNKNOWN'	'/api/token'	'REST/API'	''	'inline-script:root-url | inline-script:api-pattern'
4	'POST'	'/api/user/comment/create'	'REST/API'	'user_id, key, message'	1	'inline-script:fetch'
5	'UNKNOWN'	'/api/user/comment/create'	'REST/API'	'user_id, key, message'	2	'inline-script:root-url | inline-script:api-pattern'
6	'UNKNOWN'	'/api/user/comment/hide'	'REST/API'	''	'inline-script:root-url | inline-script:api-pattern'
7	'UNKNOWN'	'/api/user/comment/unhide'	'REST/API'	''	'inline-script:root-url | inline-script:api-pattern'
8	'GET'	'/logout'	'Authentication'	''	'html:a | html:href'
9	'GET'	'/upload/cybersechaxZine'	'File'	''	'html:a | html:href'
10	'GET'	'/upload/DefconGooNLOG'	'File'	''	'html:a | html:href'
11	'GET'	'/upload/DefconGooNLOG'	'File'	''	'html:a | html:href'
12	'GET'	'/upload/GabrielakaTrojan'	'File'	''	'html:a | html:href'
13	'GET'	'/upload/HACKFORUMSOWNER'	'File'	''	'html:a | html:href'
14	'GET'	'/upload/InfamousHaze'	'File'	''	'html:a | html:href'
15	'GET'	'/upload/KiwifarmsOwnerOwned'	'File'	''	'html:a | html:href'
16	'GET'	'/upload/KTOWNED'	'File'	''	'html:a | html:href'
17	'GET'	'/upload/NacashOwned'	'File'	''	'html:a | html:href'
18	'GET'	'/upload/NyanBB'	'File'	''	'html:a | html:href'
19	'GET'	'/upload/RyanDisney96Stenwick'	'File'	''	'html:a | html:href'
20	'GET'	'/upload/RyanLizardSquad'	'File'	''	'html:a | html:href'
21	'GET'	'/upload/RyanStenwickLOL'	'File'	''	'html:a | html:href'
22	'GET'	'/upload/Sa7anREAL'	'File'	''	'html:a | html:href'
23	'GET'	'/upload/SamGurvinderBhangu'	'File'	''	'html:a | html:href'
24	'GET'	'/upload/ShanePatrickLilly'	'File'	''	'html:a | html:href'
25	'GET'	'/upload/TheDeathofKEEMSTAR'	'File'	''	'html:a | html:href'
26	'GET'	'/upload/TryhackmeHackedel84ezine'	'File'	''	'html:a | html:href'
27	'GET'	'/upload/urharmlessOwned'	'File'	''	'html:a | html:href'
28	'GET'	'/upload/VileDotSHOwner'	'File'	''	'html:a | html:href'
29	'GET'	'/upload/WilliamJamesHollamonowned'	'File'	''	'html:a | html:href'
30	'GET'	'/upload/WoohTheGodakaZacharyDShelton'	'File'	''	'html:a | html:href'
31	'GET'	'/upload/ZacharyEvanEmswiler'	'File'	''	'html:a | html:href'
32	'GET'	'/upload/zeekill'	'File'	''	'html:a | html:href'
33	'GET'	'/upload'	'File'	''	'html:a | html:href'
Array(33)
```

Enjoy
