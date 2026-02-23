# web/Tiny SQL
> I got tired of dealing with SQL injection so I made my own SQL language.

Source provided:
```
App.py

import tinysql
from flask import Flask, request, render_template, session, redirect, url_for
from werkzeug import exceptions
from secrets import token_hex
import os
import time

TDB_HOST = os.environ.get('TDB_HOST', 'db')
TDB_PORT = int(os.environ.get('TDB_PORT', 1234)) 
FLAG = os.environ['FLAG'] if 'FLAG' in os.environ else 'bkctf{test_flag}'

app = Flask(__name__)
app.config['SECRET_KEY'] = token_hex(32)
conn = None

def create_connection():
    conn = None
    try:
        conn = tinysql.Connection(TDB_HOST, TDB_PORT)
        return conn
    except:
        raise Exception('connection failed')
    return conn

@app.route('/index')
@app.route('/forum')
@app.route('/', methods=['GET'])
def index():
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET': return render_template('login.html')

    # user lookup syntax `S:[user]:[pass]#comment`
    # results = [id, user, pass]
    code, results = conn.query('S:' + request.form['user'] +  ':' + request.form['pass'])
    if code == 'e':
        return render_template('500.html'), 500
    if len(results) != 3: return render_template('login.html'), 403
    session['username'] = results[1]
    print (session['username'])
    return redirect('/')
    

@app.route('/close', methods=['GET'])
def close():
    conn.close()
    return 'closed'

@app.route('/forum/post/<int:post_id>', methods=['GET'])
def forum(post_id):
    if not 'username' in session or session['username'] is None:
        return redirect('/login')
    post_id = post_id % 4
    # id lookup syntax `S:[id]#comment`
    code, author = conn.query('S:' + str(post_id))
    match post_id:
        case 0:
            title = 'how 2 beat da game?'
            description = 'i dont get how 2 beat the game. its so hard'
        case 1:
            title = 'the game makes computer mean??'
            description = 'every time i run the game it says "all your base are belong to us. please send us $40 in cash (bitcoin hasnt been invented yet)" \n\nbtw i downloaded it from the pirate bay if thats relevant.'
        case 2:
            title = 'an analytical discussion on the DOOM franchise'
            description = 'in this essay i will explain the way the playstyle of doom 2 while on the surface similar to the original fails to convey the antiwar story hidden in the subtext.'
        case 3:
            title = 'flag'
            description = FLAG
        case _:
            title = "this should not be seen"
            description = "this should relaly not be sean"
    return render_template('post.html', title=title, desc=description, author=author[1])

@app.errorhandler(exceptions.NotFound)
def handle_exception(e):
    if e.code == 404: return render_template('404.html'), 404
    else: return "i didnt make an error page for this", e.code


if __name__ == '__main__':
    max_retries = 10
    for i in range(max_retries):
        try:
            conn = create_connection()
            conn.query(f'I:bob:{token_hex(4)}')
            conn.query(f'I:jo:{token_hex(5)}')
            conn.query(f'I:ani:{token_hex(4)}')
            conn.query(f'I:john_r:{token_hex(3)}')
            break
        except Exception as e:
            if i < max_retries - 1:
                print(f"Connection failed, retrying in 2s... ({i+1}/{max_retries})")
                time.sleep(2)
            else:
                raise
    try:
        app.run(host="0.0.0.0", debug=False, port=3000)
    except KeyboardInterrupt:
        close()
        print ("Shutting down")
```
## Background
The creator of this challenge made their own SQL-like language syntax in an attempt to avoid SQLi. 

Reading the backend code, we have a few hints at the general syntax of their language.

It seems like each action follows a multi-part declaration using semicolons.

(Command):(data):(data)

E.x:

I:bob:token_hex(4) - inserts the user "bob" into the table with a random 4 digit password

S:user:1234 - selects a user from the table with a password of 1234.

#comments a line out

Reading the backend code, it seems like we have 4 users inserted: bob, jo, ani, and john_r. Each with their own unique password.

To view the flag, all we have to do is log in as any user.

While the token_hex(3) is definitely brute-forcable, the challenge explicitly states that we cannot do so. Thus we will have to look for 
alternate ways to get into an account.

A simple way to perform an SQLi is to just login as a user, then comment the rest of the line out (E.x: admin'--)

What we can do here is the same principle. We can log in using the user "bob" by simply inserting:

username: "bob#"

Password: "any_string"

And then we get the flag:
