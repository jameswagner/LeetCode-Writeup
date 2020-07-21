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

Following and unfollowing involve adding or removing the followed person to the follower's Set of users. Adding a user more than once to a Set or attempting to remove a user not present in the Set are already no-op operations so we don't need to explicitly check for these cases. In the case of following we also need to add the follower to the set of users if they are not already present. 
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

Get news feed involves the most computation. With each user's Tweets being sorted in a List by timestamp, getting the most 10 most recent has some similarity with [Merge k Sorted lists](https://leetcode.com/problems/merge-k-sorted-lists/). We go with a Priority Queue (Heap) data structure to order the relevant Tweets by timestamp.

In this implementation, the Tweets are not directly put on the Priority Queue, but rather a two element int array with the first element being the id of the User who posted the Tweet and the second the index of the element. This allows comparison of Tweets by timestamp while at the same time allowing quick lookup of the User of the Tweet when it is polled from the Queue, so the next most recent Tweet of the user can be easily found. An alternative could have been to include Id of the User in the Tweet data model itself but this would lead to a two-way association / cyclic dependency between Tweet and User which is avoided by not including this reference, given that User already includes a list of Tweets in its data structure. 


```
    /** Retrieve the 10 most recent tweet ids in the user's news feed. Each item in the news feed must be posted by users who the user followed or by the user herself. Tweets must be ordered from most recent to least recent. */
    public List<Integer> getNewsFeed(int userId) {
        List<Integer> newsFeed = new ArrayList(10);
        if(!users.containsKey(userId))
        {
            return newsFeed;
        }
        User user = users.get(userId);
        PriorityQueue<int[]> tweetPQ = new PriorityQueue<int[]>((a,b) -> users.get(b[0]).tweets.get(b[1]).timestamp - users.get(a[0]).tweets.get(a[1]).timestamp); // Priority Queue with most recent tweet
```

After initialization, we go through each followee and then the User, for each one adding their most recent Tweet (if any) to the Priority Queue

```
        Set<Integer> followees = user.following; // Get all followees
        
        for(Integer followeeId:followees) { 
            if(this.users.containsKey(followeeId)) {
                List<Tweet> tweets = users.get(followeeId).tweets;
                if(tweets.size()>0) { 
                  tweetPQ.offer(new int[]{followeeId,tweets.size()-1}); // add followee's most recent tweet to PQ
                }
            }
        }
        
        List<Tweet> tweets =  users.get(userId).tweets; // get their list
        if(tweets.size()>0) { // list should not be empty with this API
            tweetPQ.offer(new int[]{userId,tweets.size()-1}); // add user's most recent tweet to PQ
        }
```

Now we get the most recent Tweet from the Queue, add it to our return List, and add the next most recent by the User, (if any). Repeat 10 times or until we run out of eligible Tweets. Then the news feed is ready to return.
```
        while(!tweetPQ.isEmpty() && newsFeed.size() < 10)  {
            int[] tweetIndices = tweetPQ.poll();
            Tweet tweet = users.get(tweetIndices[0]).tweets.get(tweetIndices[1]); //Look up the Tweet by User and index.
            newsFeed.add(tweet.id);
            Integer followeeId = tweetIndices[0];
            
            
         // decrement List index and add back to the PriorityQueue, if there are still more   
            if(tweetIndices[1]>0) {
                tweetIndices[1]--;
                tweetPQ.offer(tweetIndices);
            }
        }
        return newsFeed;
    }
```
Complexity
1) Twitter overall consists of a HashMap of Users with each being a List of Tweets, so these data structures will both grow linearly. 

2) Posting a Tweet - Constant time appending to List of User's Tweets 

3) Follow and Unfollow - Assuming constant time lookup of the followee's id in the follower's HashSet, these are both constant time operations.

4) Newsfeed - If the User is following n people, the additional Priority Queue data structure will have at most 1 entry per person. Entries are fixed at arrays of 2 elements, so the space will be O(n) with respect to number of following users. Keeping the Priority Queue's heap structure such that the most recent Tweet is the next one to be polled is logarithmic for each element added or removed. The initialization will require n additions to the priority queue so this initialiation will be an O(n log n) operation. After this if we treat the 10 additional times a Tweet is removed and the User's next most recent one addded as a constant, these 10 subsequent additionals and removals will be O(log n) complexity, so the Newsfeed generation is overall O(n log n) time complexity. 
