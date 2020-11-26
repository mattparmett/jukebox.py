# Jukebox - A tiny music recommendation app

Spotify is amazing (that's a fact).

One of their best features is the "create similar playlist" function.  You can use this on
your existing playlists to have Spotify generate playlists with songs from the same artists and
artists that are similar to those artists.

I've gotten into collecting records over the past couple of years - but that music isn't in my
Spotify library unless I go through and manually save each album to a playlist.  I'm lazy and I don't
want to go through and save all my albums to Spotify, so I can't use the "create similar playlist"
function on my record collection.

I use a really cool app called [Discogs](https://discogs.com) to track the records in my collection
(they also have a cool "wantlist" feature that is like an Amazon wishlist for records).  Since all
my records are catalogued on their app, I had an idea.  Could I use python to get a list of all
the artists in my Discogs collection and generate a Spotify playlist with songs from similar artists?

Of course, the answer is yes...

Python has pretty great packages for interfacing with both the Discogs and Spotify APIs.  For
Discogs, I used [discogs_client](https://github.com/joalla/discogs_client) and for Spotify, I
used [spotipy](https://github.com/plamere/spotipy).

## 1. Register for API Access

In order to get API access, I had to register on the Discogs and Spotify developer sites.  On
Discogs, I was able to get an access token for my own account without registering a full application.
  On Spotify, I had to register my application, and then get an access token via their API, which
 I did using an application called [Postman](https://getpostman.com).

## 2. The Script

Let's import the libraries we'll need:

```python

import random
import discogs_client as dc
import spotipy as sp
```

First, we'll need to retrieve a list of all the artists in my record collection.  To do that,
let's log in to Discogs and retrieve the list of records in my collection:

```python
# Get artists in my record collection
discogs_agent = dc.Client('JukeboxAppUWPCE/0.1', user_token=DISCOGS_ACCESS_TOKEN)
discogs_user = discogs_agent.identity()
record_collection = discogs_user.collection_folders[1].releases
```

`record_collection` will be a list of records, each of which is a dict containing information
about that specific album.  Let's process that list to get a set of all the artists in my
collection.  (I'm using a set here to avoid duplicate artist entries.)

```python
artists = {record.data['basic_information']['artists'][0]['name'].split(" (")[0]
           for record in record_collection}
```

Now, let's log in to Spotify:

```python
spotify_agent = sp.Spotify(auth=SPOTIFY_ACCESS_TOKEN)
spotify_id = spotify_agent.current_user()['id']
```

Thinking ahead - in order to add tracks to a playlist, I'm going to need each track's Spotify
ID.  Let's make a set to hold those IDs (again, using a set to avoid duplicates):

```python
track_ids = set()
```

Now, we're going to have to loop through each artist in my collection and query the Spotify API
to find artists in Spotify's library that are similar to that artist.  Since there could be an
arbitrary number of artists in my record collection, I'm going to take a random sample of 10
artists to use for this script.  For each artist, our first step will be to get that artist's
Spotify ID to use in future requests.

```python
for artist in random.sample(artists, 10):
    print("Getting artists similar to: ", artist)
    artist_data = spotify_agent.search(q='artist:' + artist, type='artist')
    try:
        artist_id = artist_data['artists']['items'][0]['id']
    except IndexError:
        continue # we couldn't find the artist on Spotify
```

Let's query Spotify for similar artists:

```python
    similar_artists = spotify_agent.artist_related_artists(artist_id)['artists']
    try:
        random_similar_artists = random.sample(similar_artists, 5)
    except ValueError:
        continue # we didn't get at least 5 similar artists from Spotify
```

For each similar artist, I want to get some tracks to put into my new playlist.  I'm going to
query Spotify for a list of the artist's top tracks and add a random track from that list to my
`track_ids` set.

```python
    for similar_artist in random_similar_artists:
        print("Found similar artist: ", similar_artist['name'])
        similar_artist_top_tracks = spotify_agent.artist_top_tracks(similar_artist['id'])['tracks']
        track_ids.update([track['id'] for track in random.sample(similar_artist_top_tracks, 1)])
```

So now we've got our list of track IDs - let's add them to a new playlist:

```python
new_playlist = spotify_agent.user_playlist_create(
                user=spotify_id,
                name='Jukebox Recommendations',
                public=False,
                collaborative=False,
                description="Playlist of automatically generated song recommendations."
                )
spotify_agent.user_playlist_add_tracks(user=spotify_id, playlist_id=new_playlist['id'],
                                       tracks=track_ids, position=None)
```