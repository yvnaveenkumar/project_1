
# youtube-data-harvesting-and-warehousing

from googleapiclient.discovery import build
import pymongo
import psycopg2
import pandas as pd
import streamlit as st

# # Api Key

def Api_connect():
    api_id="AIzaSyDVQwiB4GMecMbiN3spCnWJIiZc34qzxO8"
    api_service_name="youtube"
    api_version="v3"
    
    youtube=build(api_service_name,api_version,developerKey=api_id)
    
    return youtube
youtube=Api_connect()

# # Get Channel Info 

def get_channel_id(channel_id):
    request=youtube.channels().list(
                    part="snippet,ContentDetails,statistics",
                    id=channel_id
    
    )
    response=request.execute()
    for i in response["items"]:
        data=dict(Channel_Name=i["snippet"]["title"],
                 Channel_Id=i["id"],
                 Subscribers=i["statistics"]["subscriberCount"],
                 Views=i["statistics"]["viewCount"],
                 Total_Videos=i["statistics"]["videoCount"],
                 Channel_Description=i["snippet"]["description"],
                 Playlist_Id=i["contentDetails"]["relatedPlaylists"]["uploads"])
    return data

# # Get Video Ids

def get_video_ids(channel_id):
    video_ids=[]
    response=youtube.channels().list(id=channel_id,
                                    part="contentDetails").execute()
    Playlist_Id=response["items"][0]["contentDetails"]["relatedPlaylists"]["uploads"]
    
    next_page_token=None
    
    while True:
        response1=youtube.playlistItems().list(
                                        part="snippet",
                                        playlistId=Playlist_Id,
                                        maxResults=50,
                                        pageToken=next_page_token).execute()
        for i in range(len(response1["items"])):
            video_ids.append(response1["items"][i]["snippet"]["resourceId"]["videoId"])
        next_page_token=response1.get("nextPageToken")
        
        if next_page_token is None:
            break
    return video_ids

# # Get Video Info

def get_video_info(video_ids):
    video_data=[]
    for video_id in video_ids:
        request=youtube.videos().list(
                    part="snippet,ContentDetails,statistics",
                    id=video_id
                )
        response=request.execute()

        for item in response["items"]:
            data=dict(channel_Name=item["snippet"]["channelTitle"],
                     channel_Id=item["snippet"]["channelId"],
                     video_Id=item["id"],
                     Title=item["snippet"]["title"],
                     Tags=item["snippet"].get("tags"),
                     Thumbnail=item["snippet"]["thumbnails"]["default"]["url"],
                     Description=item["snippet"].get("description"),
                     puplished_Date=item["snippet"]["publishedAt"],
                     Duration=item["contentDetails"]["duration"],
                     Views=item["statistics"].get("viewCount"),
                     Likes=item["statistics"].get("likeCount"),
                     comments=item["statistics"].get("commentCount"),
                     Favorite_Count=item["statistics"]["favoriteCount"],
                     Defintion=item["contentDetails"]["definition"],
                     caption_status=item["contentDetails"]["caption"]
                     )
            video_data.append(data)
    return video_data
        
# # Get Comment Info 

def get_comment_info(video_ids):
    Comment_data=[]
    try:
        for video_id in video_ids:
            request=youtube.commentThreads().list(
                part="snippet",
                videoId=video_id,
                maxResults=50
            )
            response=request.execute()
            
            for item in response["items"]:
                data=dict(Comment_id=item["snippet"]["topLevelComment"]["id"],
                         Video_Id=item["snippet"]["topLevelComment"]["snippet"]["videoId"],
                         Comment_Text=item["snippet"]["topLevelComment"]["snippet"]["textDisplay"],
                         Comment_Author=item["snippet"]["topLevelComment"]["snippet"]["authorDisplayName"],
                         Comment_Published=item["snippet"]["topLevelComment"]["snippet"]["publishedAt"])
                Comment_data.append(data)
    except:
        pass
    return Comment_data

# # Get Playlist Details

def get_Playlist_details(channel_id):
    next_page_token=None
    All_data=[]
    while True:
        request=youtube.playlists().list(
            part="snippet,contentDetails",
            channelId=channel_id,
            maxResults=50,
            pageToken=next_page_token
        )
        response=request.execute()
        
        for item in response["items"]:
            data=dict(Playlist_Id=item["id"],
                     Title=item["snippet"]["title"],
                     Channel_Id=item["snippet"]["channelId"],
                     channel_Name=item["snippet"]["channelTitle"],
                     PublishedAt=item["snippet"]["publishedAt"],
                     video_count=item["contentDetails"]["itemCount"])
            All_data.append(data)
            
        next_page_token=response.get("nextPageToken")
        if next_page_token is None:
            break
    return All_data

# # Connect to mongodb

client=pymongo.MongoClient("mongodb://localhost:27017/")
db=client["youtube_data_harvesting"]

client

# # Data Transfer TO Mongodb

def channel_details(channel_id):
    ch_details=get_channel_id(channel_id)
    pl_details=get_Playlist_details(channel_id)
    vi_ids=get_video_ids(channel_id)
    vi_details=get_video_info(vi_ids)
    com_details=get_comment_info(vi_ids)
    
    coll_1=db["channel_details"]
    coll_1.insert_one({"channel_information":ch_details,"playlist_information":pl_details,
                         "video_information":vi_details,"video_id_information":vi_ids,
                         "comment_information":com_details})
    return "uploaded!"

# # Channel Table Creation in Postgresql

def channel_table():
    mydb= psycopg2.connect(
            host="localhost",
            user="postgres",
            password="naveenyv5943",
            database="mongodatabase",
            port="5432"
            )
    cursor=mydb.cursor()

    drop_query='''drop table if exists channels'''
    cursor.execute(drop_query)
    mydb.commit()

    try:
        create_query='''create table if not exists channels(Channel_Name varchar(80),
                                                                Channel_Id varchar(80) primary key,
                                                                Subscribers bigint,
                                                                Views bigint,
                                                                Total_Videos int,
                                                                Channel_Description text,
                                                                Playlist_Id varchar(80))'''

        cursor.execute(create_query)
        mydb.commit()
    except:
        print("channels table already created")


    ch_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for ch_data in coll_1.find({},{"_id":0,"channel_information":1}):
        ch_list.append(ch_data["channel_information"])
    df=pd.DataFrame(ch_list)

    for index,row in df.iterrows():
        insert_query='''insert into channels(Channel_Name,
                                                 Channel_Id,
                                                 Subscribers,
                                                 Views,
                                                 Total_Videos,
                                                 Channel_Description,
                                                 Playlist_Id)

                                                 values(%s,%s,%s,%s,%s,%s,%s)'''

        values=(row["Channel_Name"],
                    row["Channel_Id"],
                    row["Subscribers"],
                    row["Views"],
                    row["Total_Videos"],
                    row["Channel_Description"],
                    row["Playlist_Id"])  

        try:
                cursor.execute(insert_query,values)
                mydb.commit()
        except:
                print("table already created")

# # Playlist Table Creation  In  Postgresql

def playlist_table():
    mydb= psycopg2.connect(
                host="localhost",
                user="postgres",
                password="naveenyv5943",
                database="mongodatabase",
                port="5432"
                )
    cursor=mydb.cursor()
    
    drop_query='''drop table if exists playlists'''
    cursor.execute(drop_query)
    mydb.commit()


    create_query='''create table if not exists playlists(Playlist_Id varchar(100) primary key,
                                                                    Title varchar(100),
                                                                    Channel_Id varchar(100),
                                                                    Channel_Name varchar(100),
                                                                    PublishedAt timestamp,
                                                                    video_count int
                                                                    )'''

    cursor.execute(create_query)
    mydb.commit()

    pl_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for pl_data in coll_1.find({},{"_id":0,"playlist_information":1}):
        for i in range(len(pl_data["playlist_information"])):
            pl_list.append(pl_data["playlist_information"][i])
    df1=pd.DataFrame(pl_list)

    for index,row in df1.iterrows():
            insert_query='''insert into playlists(Playlist_Id,
                                                     Title,
                                                     Channel_Id,
                                                     channel_Name,
                                                     PublishedAt,
                                                     video_count
                                                     )

                                                     values(%s,%s,%s,%s,%s,%s)'''

            values=(row["Playlist_Id"],
                        row["Title"],
                        row["Channel_Id"],
                        row["channel_Name"],
                        row["PublishedAt"],
                        row["video_count"])  


            cursor.execute(insert_query,values)
            mydb.commit()

# # Video Table Creation In Postgresql

def video_table():
    mydb= psycopg2.connect(
                    host="localhost",
                    user="postgres",
                    password="naveenyv5943",
                    database="mongodatabase",
                    port="5432"
                    )
    cursor=mydb.cursor()

    drop_query='''drop table if exists videos'''
    cursor.execute(drop_query)
    mydb.commit()


    create_query='''create table if not exists videos(channel_Name varchar(100),
                                                          channel_Id varchar(100),
                                                          video_Id varchar(30) primary key,
                                                          Title varchar(150),
                                                          Tags text,
                                                          Thumbnail varchar(200),
                                                          Description text,
                                                          puplished_Date timestamp,
                                                          Duration interval,
                                                          Views bigint,
                                                          Likes bigint,
                                                          comments int,
                                                          Favorite_Count int,
                                                          Defintion varchar(30),
                                                          caption_status varchar(60))'''

    cursor.execute(create_query)
    mydb.commit()

    vi_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for vi_data in coll_1.find({},{"_id":0,"video_information":1}):
        for i in range(len(vi_data["video_information"])):
            vi_list.append(vi_data["video_information"][i])
    df2=pd.DataFrame(vi_list)

    for index,row in df2.iterrows():
                insert_query='''insert into videos(channel_Name,
                                                   channel_Id,
                                                   video_Id,
                                                   Title,
                                                   Tags,
                                                   Thumbnail,
                                                   Description,
                                                   puplished_Date,
                                                   Duration,
                                                   Views,
                                                   Likes,
                                                   comments,
                                                   Favorite_Count,
                                                   Defintion,
                                                   caption_status
                                                         )

                                                   values(%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)'''

                values=(row["channel_Name"],
                            row["channel_Id"],
                            row["video_Id"],
                            row["Title"],
                            row["Tags"],
                            row["Thumbnail"],
                            row["Description"],
                            row["puplished_Date"],
                            row["Duration"],
                            row["Views"],
                            row["Likes"],
                            row["comments"],
                            row["Favorite_Count"],
                            row["Defintion"],
                            row["caption_status"]) 


                cursor.execute(insert_query,values)
                mydb.commit()

# # Comment table Creation In Postgresql

def comments_table():
    mydb= psycopg2.connect(
                        host="localhost",
                        user="postgres",
                        password="naveenyv5943",
                        database="mongodatabase",
                        port="5432"
                        )
    cursor=mydb.cursor()

    drop_query='''drop table if exists comments'''
    cursor.execute(drop_query)
    mydb.commit()


    create_query='''create table if not exists comments(Comment_id varchar(100) primary key,
                                                       Video_Id varchar(50),
                                                       Comment_Text text,
                                                       Comment_Author varchar(150),
                                                       Comment_Published timestamp
                                                       )'''

    cursor.execute(create_query)
    mydb.commit()



    com_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for com_data in coll_1.find({},{"_id":0,"comment_information":1}):
        for i in range(len(com_data["comment_information"])):
             com_list.append(com_data["comment_information"][i])
    df3=pd.DataFrame(com_list)


    for index,row in df3.iterrows():
                insert_query='''insert into comments(Comment_id,
                                                     Video_Id,
                                                     Comment_Text,
                                                     Comment_Author,
                                                     Comment_Published
                                                     )

                                                    values(%s,%s,%s,%s,%s)'''

                values=(row["Comment_id"],
                        row["Video_Id"],
                        row["Comment_Text"],
                        row["Comment_Author"],
                        row["Comment_Published"])  


                cursor.execute(insert_query,values)
                mydb.commit()

def tables():
    channel_table()
    playlist_table()
    video_table()
    comments_table()
    
    return "table created successfully"

result = tables()

# # Show Tables In Streamlit App

def show_channels_table():
    ch_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for ch_data in coll_1.find({},{"_id":0,"channel_information":1}):
        ch_list.append(ch_data["channel_information"])
    df=st.dataframe(ch_list)
    
    return df

def show_playlists_table():
    pl_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for pl_data in coll_1.find({},{"_id":0,"playlist_information":1}):
        for i in range(len(pl_data["playlist_information"])):
            pl_list.append(pl_data["playlist_information"][i])
    df1=st.dataframe(pl_list)
    
    return df1

def show_videos_table():
    vi_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for vi_data in coll_1.find({},{"_id":0,"video_information":1}):
        for i in range(len(vi_data["video_information"])):
            vi_list.append(vi_data["video_information"][i])
    df2=st.dataframe(vi_list)

    return df2

def show_comments_table():
    com_list=[]
    db=client["youtube_data_harvesting"]
    coll_1=db["channel_details"]
    for com_data in coll_1.find({},{"_id":0,"comment_information":1}):
        for i in range(len(com_data["comment_information"])):
            com_list.append(com_data["comment_information"][i])
    df3=st.dataframe(com_list)

    return df3

with st.sidebar:
    st.title(":red[YOUTUBE DATA HARVESTING AND WAREHOUSING]")
    st.header("SKILL TAKE AWAY")
    st.caption('Python scripting')
    st.caption("Data Collection")
    st.caption("MongoDB")
    st.caption("API Integration")
    st.caption(" Data Managment using MongoDB and SQL")
    
channel_id = st.text_input("Enter the Channel id")
channels = channel_id.split(',')
channels = [ch.strip() for ch in channels if ch]

if st.button("Collect and Store data"):
    for channel in channels:
        ch_ids = []
        db = client["youtube_data_harvesting"]
        coll1 = db["channel_details"]
        for ch_data in coll1.find({},{"_id":0,"channel_information":1}):
            ch_ids.append(ch_data["channel_information"]["Channel_Id"])
        if channel in ch_ids:
            st.success("Channel details of the given channel id: " + channel + " already exists")
        else:
            output = channel_details(channel)
            st.success(output)
            
if st.button("Migrate to SQL"):
    display = tables()
    st.success(display)
    
show_table = st.radio("SELECT THE TABLE FOR VIEW",(":green[channels]",":orange[playlists]",":red[videos]",":blue[comments]"))

if show_table == ":green[channels]":
    show_channels_table()
elif show_table == ":orange[playlists]":
    show_playlists_table()
elif show_table ==":red[videos]":
    show_videos_table()
elif show_table == ":blue[comments]":
    show_comments_table()

#SQL connection
mydb= psycopg2.connect(
                    host="localhost",
                    user="postgres",
                    password="naveenyv5943",
                    database="mongodatabase",
                    port="5432"
                    )
cursor=mydb.cursor()

question = st.selectbox(
    'Please Select Your Question',
    ('1. All the videos and the Channel Name',
     '2. Channels with most number of videos',
     '3. 10 most viewed videos',
     '4. Comments in each video',
     '5. Videos with highest likes',
     '6. likes of all videos',
     '7. views of each channel',
     '8. videos puplished in the year 2022',
     '9. average duration of all videos in each channel',
     '10. videos with highest number of comments'))

if question == '1. All the videos and the Channel Name':
    query1 = "select Title as videos, Channel_Name as ChannelName from videos;"
    cursor.execute(query1)
    mydb.commit()
    t1=cursor.fetchall()
    st.write(pd.DataFrame(t1, columns=["Video Title","Channel Name"]))

elif question == '2. Channels with most number of videos':
    query2 = "select Channel_Name as ChannelName,Total_Videos as NO_Videos from channels order by Total_Videos desc;"
    cursor.execute(query2)
    mydb.commit()
    t2=cursor.fetchall()
    st.write(pd.DataFrame(t2, columns=["Channel Name","No Of Videos"]))

elif question == '3. 10 most viewed videos':
    query3 = '''select Views as views , Channel_Name as ChannelName,Title as VideoTitle from videos 
                        where Views is not null order by Views desc limit 10;'''
    cursor.execute(query3)
    mydb.commit()
    t3 = cursor.fetchall()
    st.write(pd.DataFrame(t3, columns = ["views","channel Name","video title"]))

elif question == '4. Comments in each video':
    query4 = "select Comments as No_comments ,Title as VideoTitle from videos where Comments is not null;"
    cursor.execute(query4)
    mydb.commit()
    t4=cursor.fetchall()
    st.write(pd.DataFrame(t4, columns=["No Of Comments", "Video Title"]))

elif question == '5. Videos with highest likes':
    query5 = '''select Title as VideoTitle, Channel_Name as ChannelName, Likes as LikesCount from videos 
                       where Likes is not null order by Likes desc;'''
    cursor.execute(query5)
    mydb.commit()
    t5 = cursor.fetchall()
    st.write(pd.DataFrame(t5, columns=["video Title","channel Name","like count"]))

elif question == '6. likes of all videos':
    query6 = '''select Likes as likeCount,Title as VideoTitle from videos;'''
    cursor.execute(query6)
    mydb.commit()
    t6 = cursor.fetchall()
    st.write(pd.DataFrame(t6, columns=["like count","video title"]))

elif question == '7. views of each channel':
    query7 = "select Channel_Name as ChannelName, Views as Channelviews from channels;"
    cursor.execute(query7)
    mydb.commit()
    t7=cursor.fetchall()
    st.write(pd.DataFrame(t7, columns=["channel name","total views"]))

elif question == '8. videos puplished in the year 2022':
    query8 = '''select Title as Video_Title, Puplished_Date as VideoRelease, Channel_Name as ChannelName from videos 
                where extract(year from Puplished_Date) = 2022;'''
    cursor.execute(query8)
    mydb.commit()
    t8=cursor.fetchall()
    st.write(pd.DataFrame(t8,columns=["Name", "Video Puplished On", "ChannelName"]))

elif question == '9. average duration of all videos in each channel':
    query9 =  "SELECT Channel_Name as ChannelName, AVG(Duration) AS average_duration FROM videos GROUP BY Channel_Name;"
    cursor.execute(query9)
    mydb.commit()
    t9=cursor.fetchall()
    t9 = pd.DataFrame(t9, columns=['ChannelTitle', 'Average Duration'])
    T9=[]
    for index, row in t9.iterrows():
        channel_title = row['ChannelTitle']
        average_duration = row['Average Duration']
        average_duration_str = str(average_duration)
        T9.append({"Channel Title": channel_title ,  "Average Duration": average_duration_str})
    st.write(pd.DataFrame(T9))

elif question == '10. videos with highest number of comments':
    query10 = '''select Title as VideoTitle, Channel_Name as ChannelName, Comments as Comments from videos 
                       where Comments is not null order by Comments desc;'''
    cursor.execute(query10)
    mydb.commit()
    t10=cursor.fetchall()
    st.write(pd.DataFrame(t10, columns=['Video Title', 'Channel Name', 'NO Of Comments']))
