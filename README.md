## [Steemit Curation Flow](https://fdsteemtools.neocities.org/curation_flow.html)
### About [Curation Flow](https://fdsteemtools.neocities.org/curation_flow.html)
Steemit Curation Flow is a single web-page tool that shows the potential *curation rewards* earned based on your upvotes.

![image.png](https://cdn.utopian.io/posts/8deb03bb301fc3ba15a0ca6e59a0667d5d04image.png)

With all the upvotes that are given to posts, curation rewards are earned.
There were really good documents on calculation formula of curation rewards.
* @YabapMatt has a [tool and a really explaining post](https://steemit.com/utopian-io/@yabapmatt/curation-reward-estimation-tool) for a single post curation estimation.
This tool I used for verification of my calculations.
* @miniature-tiger has a [great post](https://steemit.com/curation/@miniature-tiger/an-illustrated-guide-to-curation-from-the-simple-to-the-complex-with-real-examples-from-past-posts-part-1) on visually explaining the curation reward system.
With the help of his tool, I had a deeper understanding and slightly differentiated @YabapMatt's formula to fit my code, giving exactly the same result.

### How to use Curation Flow?
* Go to web-page https://fdsteemtools.neocities.org/curation_flow.html
* Enter your name in the username section

![image.png](https://cdn.utopian.io/posts/28df2d3415942d513ed6a5432d88886da09fimage.png)

* Press "Calculate" button

![image.png](https://cdn.utopian.io/posts/b20888d4de8cfb4306638415cdfd706c8b46image.png)

* You will see your curation reward pay-outs and the dates in the divs.

### Code
Curation flow is using HTML and [steem.js](https://github.com/steemit/steem-js) API

* Calculate the feed-price, this is used to convert SBD payments to steem.
```
   // Get current feed price
    var feed;
    var the_voter;
    steem.api.getFeedHistory(function(err, result) {
      var feed_arr = result.current_median_history.base;
      feed = feed_arr.split(" ")[0];
    });
```
* Get all the posts the user has voted and eliminate the ones that are older than 7 days.
```
var aut = [];
      var perm = [];
      var post_date = [];

      // Get all the post data that user have upvoted
      steem.api.getAccountVotes(the_voter, function(err, result) {

        // Calculate the current time as UTC since post dates are in UTC
        var now = new Date();
        var nowUtc = new Date(now.getTime() + (now.getTimezoneOffset() * 60000));
        // Convert UTC date to timestamp
        var now_stamp = now.getTime();
        var nowUtc_stamp = nowUtc.getTime();
        // Put variable to check -7 days
        var limit = nowUtc_stamp - 24 * 60 * 60 * 1000 * 7

        for (let i = 0; i < result.length; i++) {
          // convert post date to timestamp
          var ptime = Date.parse(result[i].time);
          post_date.push(ptime);
          // Disregard if it is a downvote or if the post is older than exactly 7 days
          if ((result[i].rshares > 0) && (ptime >= limit)) {
            // form the arrays with information of suitable posts

            aut.push((result[i].authorperm).split("/")[0]);
            perm.push((result[i].authorperm).split("/")[1]);

          }
        }

        get_content(aut, perm, limit);
      });

    }
```
* Get the content of the posts that are upvoted
```
 // This function forms the content array according to the voted posts
    function get_content(aut, per, limit) {

      var result_array = [];

      var count = 0;
      for (let i = 0; i < aut.length; i++) {
        // get the content for each author and permlink
        steem.api.getContent(aut[i], per[i], function(err, result) {
          var p_date = result.created;
          var p_date_stamp = Date.parse(p_date);
          // check if the post or comment is new < 7 days
          if (p_date_stamp > limit) {
            // if fresh, fill the arrays

            result_array.push(result);
          }
          // check if the async function got all the results

          // if OK, send results for calculation
          count++;
          if (count > aut.length - 1) {
            calculate(result_array);
          }

        });

      }

    }
```
* Calculate the curation rewards
```
// function for calculation of curation rewards
    function calculate(result) {
      var votes = []
      var sbd_pay = [];
      var ratio;
      var before;
      var now;
      var p_date;
      var p_date_parsed;
      var v_date;
      var v_date_parsed;
      var penalty;
      var sbd_arr;
      var sbd;
      var tot_share;
      var temp;
      var vote = [];
      var netshares = [];
      var author = [];
      var permlink = [];
      var postdate = [];
      for (let i = 0; i < result.length; i++) {    
        before = 0;
        // get the active votes
		vote = result[i].active_votes;
        // sort the votes according to vote time
		vote.sort(compare);
        votes.push(vote);
        for (let j = 0; j < vote.length; j++) {
          tot_share = parseInt(result[i].net_rshares);
          now = before + parseInt(vote[j].rshares);
         // if the current total rshares is negative due to downvotes it must be equal to zero!
		  if (now < 0) {
            now = 0;
          }
		// make the calculation when user is the voter
          if (vote[j].voter == the_voter) {
           // formula of curation reward calculation
		    ratio = (Math.sqrt(now) - Math.sqrt(before)) / (Math.sqrt(tot_share));
            p_date = result[i].created;
            p_date_parsed = Date.parse(p_date);
            v_date = vote[j].time;
            v_date_parsed = Date.parse(v_date);
			// calculate the ratio if the post is voted before 30 minutes
            penalty = (v_date_parsed - p_date_parsed) / (30 * 60 * 1000);
            if (penalty >= 1) {
              penalty = 1;
            }			
			// if the voter is author, the penalty doesn't apply
            if (result[i].author == the_voter) {
              penalty = 1;
            }
            sbd_arr = result[i].pending_payout_value;
            sbd = sbd_arr.split(" ")[0];
           // if the post is a total downvote, no pay-out
		    if (parseInt(result[i].net_rshares) < 0) {
              sbd_pay.push(0);
            }
			// calculate the SP payment
            if (parseInt(result[i].net_rshares) >= 0) {
              sbd_pay.push((sbd * 0.25 * ratio * penalty) / feed);
            }
          }
          before = now;
         // no need for this, extra security!
		  if (before < 0) {
            before = 0;
          }
        }
       // form the arrays
	    netshares.push(parseInt(result[i].net_rshares));//this array is not used, just for check!
        author.push(result[i].author);
        var str = "https://steemit.com/@" + result[i].author + "/" + result[i].permlink;
        var lin = (result[i].permlink).substring(0, 70) + ".......";
        permlink.push(lin.link(str));
        postdate.push(result[i].cashout_time);
      }
     // send all to final function to be written in DIV
      final(permlink, sbd_pay, postdate);
    }
```
It is the sqrt formula that calculates the curation rewards
Ratio of reward

``` ratio = (Math.sqrt(now) - Math.sqrt(before)) / (Math.sqrt(tot_share));```

The penalty for voting before 30 mins

```penalty = (v_date_parsed - p_date_parsed) / (30 * 60 * 1000);```

The full code can be found [here](https://github.com/firedreamgames/steem_curation_flow/blob/master/curation_flow.html)

### How to contribute?
The tool gives the curation rewards received based on your upvotes
However, although insignificant there is an other curation you receive.If your posts are upvoted before 30 minutes, the autor receives the lost curation.
This can be added as a contrubition for much accurate result.

### Connect
@FireDream - Steemit

@firedream#3528 - Discord

