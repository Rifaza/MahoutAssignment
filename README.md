# MahoutAssignment
Bid data assignment - 219393M

Sign up for an AWS account. Start up an EMR cluster Get the MovieLens data

wget http://files.grouplens.org/datasets/movielens/ml-1m.zip
unzip ml-1m.zip
Convert ratings.dat, trade “::” for “,”, and take only the first three columns:

cat ml-1m/ratings.dat | sed 's/::/,/g' | cut -f1-3 -d, > ratings.csv

Put ratings file into HDFS:

hadoop fs -put ratings.csv /ratings.csv

Run the recommender job:

mahout recommenditembased --input /ratings.csv --output recommendations --numRecommendations 10 --outputPathForSimilarityMatrix similarity-matrix --similarityClassname SIMILARITY_COSINE

Look for the results in the part-files containing the recommendations:

        hadoop fs -ls recommendations
        hadoop fs -cat recommendations/part-r-00000 | head
You should see a lookup file that looks something like this (your recommendations will be different since they are all 5.0-valued and we are only picking ten):

User ID (Movie ID : Recommendation Strength) Tuples 
3	[2376:5.0,3168:5.0,3035:5.0,2968:5.0,2375:5.0,2243:5.0,198:5.0,594:5.0,2111:5.0,3494:5.0]
6	[3035:5.0,1912:5.0,3100:5.0,1187:5.0,2110:5.0,3034:5.0,3298:5.0,594:5.0,2572:5.0,2375:5.0]
9	[265:5.0,1449:5.0,1517:5.0,1912:5.0,1320:5.0,3168:5.0,198:5.0,3100:5.0,2112:5.0,196:5.0]
12	[592:5.0,1189:5.0,3098:5.0,3424:5.0,930:5.0,928:5.0,3498:5.0,1321:5.0,1387:5.0,596:5.0]
15	[3354:4.6306524,3793:4.5098224,3189:4.509001,589:4.500217,3755:4.479548,3949:4.4727006,3861:4.4621515,3646:4.458238,3827:4.4371557,3745:4.4361334]
18	[2376:5.0,3168:5.0,1187:5.0,2112:5.0,2243:5.0,1320:5.0,3430:5.0,3035:5.0,1584:5.0,1449:5.0]
21	[316:5.0,3785:5.0,3911:5.0,2600:5.0,3317:5.0,442:5.0,1909:5.0,3863:5.0,1527:5.0,2542:5.0]
24	[2376:5.0,2243:5.0,3035:5.0,529:5.0,2374:5.0,2375:5.0,2111:5.0,2110:5.0,265:5.0,3168:5.0]
27	[2968:5.0,3629:5.0,1253:5.0,3100:5.0,1188:5.0,3035:5.0,198:5.0,3298:5.0,594:5.0,2243:5.0]
30	[1234:5.0,903:5.0,788:5.0,1961:5.0,1036:5.0,1228:4.753679,3360:4.738523,1242:4.7331953,2871:4.6802135,3256:4.666473]

Where the first number is a user id, and the key-value pairs inside the brackets are movie-id:recommendation-strength tuples.

The recommendation strengths are at a hundred percent, or 5.0 in this case, and should work to finesse the results. This probably indicates that there are many more than ten “perfect five” recommendations for most people, so you might calculate more than the top ten or pull from deeper in the ranking to surface less-popular items.

Building a Service Next, we’ll use this lookup file in a simple web service that returns movie recommendations for any given user.

Get Twisted, and Klein and Redis modules for Python.

        sudo pip3 install twisted
        sudo pip3 install klein
        sudo pip3 install redis
Install Redis and start up the server.

        wget http://download.redis.io/releases/redis-2.8.7.tar.gz
        tar xzf redis-2.8.7.tar.gz
        cd redis-2.8.7
        make
        ./src/redis-server &
Build a web service that pulls the recommendations into Redis and responds to queries. Put the following into a file, e.g., “hello.py”

from klein import run, route
import redis
import os

# Start up a Redis instance
r = redis.StrictRedis(host='localhost', port=6379, db=0)

# Pull out all the recommendations from HDFS
p = os.popen("hadoop fs -cat recommendations/part*")

# Load the recommendations into Redis
for i in p:

  # Split recommendations into key of user id 
  # and value of recommendations
  # E.g., 35^I[2067:5.0,17:5.0,1041:5.0,2068:5.0,2087:5.0,
  #       1036:5.0,900:5.0,1:5.0,081:5.0,3135:5.0]$
  k,v = i.split('t')

  # Put key, value into Redis
  r.set(k,v)

# Establish an endpoint that takes in user id in the path
@route('/<string:id>')

def recs(request, id):
  # Get recommendations for this user
  v = r.get(id)
  return 'The recommendations for user '+id+' are '+v.decode()


# Make a default endpoint
@route('/')

def home(request):
  return 'Please add a user id to the URL, e.g. http://localhost:8083/1234n'

# Start up a listener on port 8083
run("localhost", 8083)
Start the web service.

twistd -noy hello.py &

Test the web service with user id “27”:
curl localhost:8083/27
You should see a response like this (again, your recommendations will differ):
The recommendations for user 27 are [2968:5.0,3629:5.0,1253:5.0,3100:5.0,1188:5.0,3035:5.0,198:5.0,3298:5.0,594:5.0,2243:5.0]

