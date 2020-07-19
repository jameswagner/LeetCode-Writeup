# Design Twitter

We'll start with simple Tweet and User data structures to store the relevant information. Tweets are very basic, Users contain Tweets and the ids of Users they are following.

```Java
class Tweet {
    int id;
    int timestamp;
    public Tweet(int id, int timestamp) {
        this.id = id;
        this.timestamp = timestamp;
    }
}

class User {
    int id;
    List<Tweet> tweets;
    Set<Integer> following;
    public User(int id) {
        this.id = id;
        tweets = new ArrayList<Tweet>();
        following = new HashSet<Integer>();
    }
}
```

Initializing Twitter, we will keep a set of users and a timestamp counter to provide ordering of the tweets. 
```Java
class Twitter {
    
    Map<Integer,User> users;
    int currentTimestamp;

    /** Initialize your data structure here. */
    public Twitter() {
        users = new HashMap<Integer,User>();
        currentTimestamp = 0;
    }
```

To post a tweet, add that tweet to the list of tweets by that user (also create a new user and add to the Map if the user id is not already there)
```
    /** Compose a new tweet. Add tweet to list of tweets by user*/
    public void postTweet(int userId, int tweetId) {
        users.putIfAbsent(userId, new User(userId));
        User user = users.get(userId);
        List<Tweet> tweets = user.tweets;
        tweets.add(new Tweet(tweetId, currentTimestamp++));
    }
```

Following and unfollowing involve adding or removing the followed person to the set. Adding a user more than once to a set or attempting to remove a user not present in the set are no-op operations so we don't need to explicitly check for these cases. In the case of following we also need to add the follower to the set of users if they are not already present. 
```
    /** Follower follows a followee. If the operation is invalid, it should be a no-op. */
    public void follow(int followerId, int followeeId) {
        if(followerId == followeeId) {
          return;
        }
        users.putIfAbsent(followerId, new User(followerId));
        User user = users.get(followerId);
        Set<Integer> following = user.following;
        following.add(followeeId);
    }
    
    /** Follower unfollows a followee. If the operation is invalid, it should be a no-op. */
    public void unfollow(int followerId, int followeeId) {
        if(followerId==followeeId || !users.containsKey(followerId)) {
          return;
        }
        User user = users.get(followerId);
        Set<Integer> following = user.following;
        following.remove(new Integer(followeeId));
    }
```


```
    /** Retrieve the 10 most recent tweet ids in the user's news feed. Each item in the news feed must be posted by users who the user followed or by the user herself. Tweets must be ordered from most recent to least recent. */
    public List<Integer> getNewsFeed(int userId) {
        List<Integer> newsFeed = new ArrayList(10);
        if(!users.containsKey(userId))
        {
            return newsFeed;
        }
        PriorityQueue<int[]> tweetPQ = new PriorityQueue<int[]>((a,b) -> users.get(b[0]).tweets.get(b[1]).timestamp - users.get(a[0]).tweets.get(a[1]).timestamp); // Priority Queue with most recent tweet

        Set<Integer> followees = users.get(userId).following; // Get all followees
        
        for(Integer followeeId:followees) { // get tweets of each followee
            if(this.users.containsKey(followeeId)) { // does followee have a list of tweets
                List<Tweet> tweets = users.get(followeeId).tweets;
                if(tweets.size()<1) { 
                    continue;
                }
                tweetPQ.offer(new int[]{followeeId,tweets.size()-1}); // add followee's most recent tweet to PQ
            }
        }
        
        List<Tweet> tweets =  users.get(userId).tweets; // get their list
        if(tweets.size()>0) { // list should not be empty with this API
            tweetPQ.offer(new int[]{userId,tweets.size()-1}); // add user's most recent tweet to PQ
        }
                    
        while(!tweetPQ.isEmpty() && newsFeed.size() < 10)
        {
            int[] tweetIndices = tweetPQ.poll();
            Tweet tweet = users.get(tweetIndices[0]).tweets.get(tweetIndices[1]);
            newsFeed.add(tweet.id);
            Integer followeeId = tweetIndices[0];
            
            if(tweetIndices[1]>0) // add it to the priority queue (if it exists)
            {
                tweetIndices[1]--;
                tweetPQ.offer(tweetIndices);
            }
        }
        return newsFeed;
        
    }

    

