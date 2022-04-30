---
pcx-content-type: tutorial
title: Build your own Analytics dashboard
weight: 41
---

# Build your own Analytics dashboard

In this example, we are going to see how to use the GraphQL Analytics API to build your own dashboard. This tutorial walks you through building a simple line chart for your Cloudflare zone using HTML, JavaScript, AJAX, and chart.js.

![Creating a chart with GraphQL showing zone traffic](../static/graphQL-recipe-cacheVisual.gif)

The following code will build a page with all the requirements to fetch from GraphQL and plot the cached and uncached bandwidth for the given zone. You will just need to enter your email address, API key, and your zone ID, and then click the **Fetch analytics** button. To download an example of a `CSS` file, you can click [here](/analytics/static/downloads/main.css).

## Code

{{<Aside type="note" header="Note">}}

Cloudflare's [GraphQL endpoint](https://api.cloudflare.com/client/v4/graphql) does not set any CORS headers. Add an endpoint that can proxy the requests back to the API to avoid encountering CORS errors. In the following example, this hostname is referred to as `api.yourdomain.com`.

{{</Aside>}}

```html
<!DOCTYPE html>
<html>

<head>
    <title>Line Chart</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.8.0/Chart.bundle.min.js" integrity="sha512-60KwWtZOhzgr840mc57MV8JqDZHAws3w61mhK45KsYHmhyNFJKmfg4M7/s2Jsn4PgtQ4Uhr9xItS+HCbGTIRYQ==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
    <script src="https://www.chartjs.org/samples/latest/utils.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
    <link rel="stylesheet" href="main.css">
</head>

<body>
    <div>
        <h3>
            <span>Visualise your traffic from GraphQL!</span>
        </h3>
        <br />
        <div>
            <form>
                <label for="site" class=""><span>Enter your API details:</span></label>
                <input placeholder="Email" id="email" />
                <input placeholder="API Key" id="apiKey" />
            </form>
        </div>
        <label for="site" class=""><span>Choose the zone tag you want to fetch for:</span></label>
        <div>
            <input placeholder="Zone Tag" id="zoneTag" />
        </div>
        <button id="fetch" type="button" data-testid="add-site-button">
            <p><span>Fetch analytics</span></p>
        </button>
    </div>

    <span id="error" style="color: red"></span>

    <canvas id="canvas" style="width:100%;"></canvas>
    <br />
    <br />

    <script>
        function chartInit(date, total, cached){
            var config = {
                type: 'line',
                data: {
                    labels: date,
                    datasets: [{
                        label: 'Total traffic',
                        backgroundColor: 'red',
                        borderColor: 'red',
                        data: total,
                        fill: false,
                    }, {
                        label: 'Cached traffic',
                        backgroundColor: 'blue',
                        borderColor: 'blue',
                        fill: false,
                        data: cached
                    }]
                },
                options: {
                    responsive: true,
                    title: {
                        display: true,
                        text: 'Cached vs Uncached traffic'
                    },
                    tooltips: {
                        mode: 'index',
                        intersect: false,
                    },
                    hover: {
                        mode: 'nearest',
                        intersect: true
                    },
                    scales: {
                        xAxes: [{
                            display: true,
                            scaleLabel: {
                                display: true,
                                labelString: 'Date'
                            }
                        }],
                        yAxes: [{
                            display: true,
                            scaleLabel: {
                                display: true,
                                labelString: 'Bytes'
                            }
                        }]
                    }
                }
            };
            return config;
        }

        function fetchAPI(url, email, apiKey, zoneTag) {
            const today = new Date();
            const thirtyDaysAgo = new Date(new Date().setDate(today.getDate() - 30));
            const date = thirtyDaysAgo.toISOString().split('T')[0];

            let request = new Request(url)
            const query = `query {
                viewer {
                  zones(filter: { zoneTag: $zoneTag }) {
                    httpRequests1dGroups(
                      orderBy: [date_ASC]
                      limit: 1000
                      filter: { date_gt: $date }
                    ) {
                      date: dimensions {
                        date
                      }
                      sum {
                        cachedBytes
                        bytes
                      }
                    }
                  }
                }
              }`;

            const json = { query, variables: { zoneTag, date,  } };

            let init = {
                method: 'POST',
                body: JSON.stringify(json)
            }
            request.headers.set('x-auth-key', apiKey)
            request.headers.set('x-auth-email', email)

            return fetch(request, init)
        }

        document.getElementById('fetch').addEventListener('click', async function() {
            var email = document.getElementById('email').value
            var apiKey = document.getElementById('apiKey').value
            var zoneTag = document.getElementById('zoneTag').value
            var ctx = document.getElementById('canvas').getContext('2d');
            var beforeGraph = document.getElementById('canvas')

            if (!zoneTag) {
                document.getElementById('error').innerText = 'No Zone ID set!';
                return;
            }
            
            if (email && apiKey) {
                document.getElementById('error').innerHTML = ""

                // Replace the `api.yourdomain.com` with your CORS proxy.
                // We have an example Worker here you can use: https://developers.cloudflare.com/workers/examples/cors-header-proxy/
                let response = await fetchAPI('https://api.yourdomain.com/client/v4/graphql', email, apiKey, zoneTag)
                let json = await response.json()

                if (response.status === 200) {
                    var date = []
                    var total = []
                    var cached = []
                    var array = json.data.viewer.zones[0].httpRequests1dGroups

                    for (let i = 0; i < array.length; i++) {
                        date.push(array[i].date.date)
                        total.push(array[i].sum.bytes)
                        cached.push(array[i].sum.cachedBytes)
                    }

                    window.myLine = new Chart(ctx, chartInit(date, total, cached));
                } else {
                    document.getElementById('error').innerText = 'error: \n'+ json
                    ctx.clearRect(0, 0, canvas.width, canvas.height);
                }
            } else {
                document.getElementById('error').innerText = "Please fill the form with your key and email"
                ctx.clearRect(0, 0, canvas.width, canvas.height);
            }
        });
    </script>
</body>
</html>
```
