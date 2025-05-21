<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Blogger Admin Dashboard</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 0; padding: 0;
      display: flex;
      height: 100vh;
    }
    nav.sidebar {
      width: 250px;
      background: #2c3e50;
      color: white;
      padding: 20px;
      box-sizing: border-box;
    }
    nav.sidebar h2 {
      margin-top: 0;
      font-weight: normal;
    }
    nav.sidebar ul {
      list-style: none;
      padding: 0;
    }
    nav.sidebar ul li {
      margin: 15px 0;
      cursor: pointer;
    }
    nav.sidebar ul li:hover {
      text-decoration: underline;
    }
    main.content {
      flex: 1;
      padding: 20px;
      box-sizing: border-box;
      overflow-y: auto;
    }
    .section {
      margin-bottom: 40px;
    }
    textarea, input[type="text"] {
      width: 100%;
      font-family: monospace;
      font-size: 14px;
      margin-bottom: 10px;
      padding: 8px;
      box-sizing: border-box;
    }
    button {
      padding: 10px 15px;
      background: #2980b9;
      color: white;
      border: none;
      cursor: pointer;
      margin-top: 10px;
    }
    button:hover {
      background: #3498db;
    }
    #signin-button {
      margin-bottom: 20px;
    }
  </style>
</head>
<body>

  <nav class="sidebar">
    <h2>Blogger Admin</h2>
    <ul>
      <li onclick="showSection('posts')">Posts Management</li>
      <li onclick="showSection('newpost')">Add New Post</li>
    </ul>
  </nav>

  <main class="content">

    <div id="signin-section">
      <button id="signin-button">Sign In with Google</button>
      <button id="signout-button" style="display:none;">Sign Out</button>
      <p id="status">Not signed in</p>
    </div>

    <div id="posts" class="section" style="display:none;">
      <h3>Posts List</h3>
      <button onclick="listPosts()">Refresh Posts</button>
      <ul id="postList"></ul>
    </div>

    <div id="newpost" class="section" style="display:none;">
      <h3>Add New Post</h3>
      <input type="text" id="postTitle" placeholder="Post Title" />
      <textarea id="postContent" rows="8" placeholder="Post Content (HTML allowed)"></textarea>
      <button onclick="createPost()">Create Post</button>
      <p id="postStatus"></p>
    </div>

  </main>

<script src="https://apis.google.com/js/api.js"></script>
<script>
  const CLIENT_ID = '702378759612-0ko1q1qud0dkqjk65cclau4mcasn8iku.apps.googleusercontent.com';
  const API_KEY = 'AIzaSyAzX-zbOyJt8oaZbOv7QvVPRTxG30Bv2iA';
  const BLOG_ID = '1736694429878052484';
  const DISCOVERY_DOCS = ["https://www.googleapis.com/discovery/v1/apis/blogger/v3/rest"];
  const SCOPES = 'https://www.googleapis.com/auth/blogger';

  const signinButton = document.getElementById('signin-button');
  const signoutButton = document.getElementById('signout-button');
  const statusText = document.getElementById('status');

  let gapiInited = false;
  let gisInited = false;
  let tokenClient;

  function showSection(id) {
    document.querySelectorAll('.section').forEach(s => s.style.display = 'none');
    if(id) document.getElementById(id).style.display = 'block';
  }

  function updateSigninStatus(isSignedIn) {
    if (isSignedIn) {
      statusText.textContent = 'Signed in';
      signinButton.style.display = 'none';
      signoutButton.style.display = 'inline-block';
      showSection('posts');
      listPosts();
    } else {
      statusText.textContent = 'Not signed in';
      signinButton.style.display = 'inline-block';
      signoutButton.style.display = 'none';
      showSection(null);
      document.getElementById('postList').innerHTML = '';
    }
  }

  // Load GAPI and initialize client
  function gapiLoaded() {
    gapi.load('client', initializeGapiClient);
  }

  async function initializeGapiClient() {
    await gapi.client.init({
      apiKey: API_KEY,
      discoveryDocs: DISCOVERY_DOCS,
    });
    gapiInited = true;
    maybeEnableButtons();
  }

  // Load Google's Identity Services
  function gisLoaded() {
    tokenClient = google.accounts.oauth2.initTokenClient({
      client_id: CLIENT_ID,
      scope: SCOPES,
      callback: '', // defined later
    });
    gisInited = true;
    maybeEnableButtons();
  }

  function maybeEnableButtons() {
    if (gapiInited && gisInited) {
      signinButton.disabled = false;
    }
  }

  signinButton.onclick = () => {
    tokenClient.callback = async (resp) => {
      if (resp.error !== undefined) {
        throw (resp);
      }
      updateSigninStatus(true);
    };

    if (!gapi.client.getToken()) {
      // Prompt the user to select a Google Account and ask for consent to share their data
      tokenClient.requestAccessToken({ prompt: 'consent' });
    } else {
      // Skip display of account chooser and consent dialog for an existing token
      tokenClient.requestAccessToken({ prompt: '' });
    }
  };

  signoutButton.onclick = () => {
    const token = gapi.client.getToken();
    if (token) {
      google.accounts.oauth2.revoke(token.access_token);
      gapi.client.setToken('');
      updateSigninStatus(false);
    }
  };

  async function listPosts() {
    const response = await gapi.client.blogger.posts.list({
      blogId: BLOG_ID,
      maxResults: 10,
      fetchBodies: true,
      status: 'live'
    });
    const posts = response.result.items || [];
    const listElem = document.getElementById('postList');
    if (posts.length === 0) {
      listElem.innerHTML = '<li>No posts found</li>';
      return;
    }
    listElem.innerHTML = '';
    posts.forEach(post => {
      const li = document.createElement('li');
      li.textContent = post.title;
      listElem.appendChild(li);
    });
  }

  async function createPost() {
    const title = document.getElementById('postTitle').value.trim();
    const content = document.getElementById('postContent').value.trim();
    const statusElem = document.getElementById('postStatus');

    if (!title || !content) {
      statusElem.textContent = 'Title and content are required.';
      return;
    }

    try {
      const response = await gapi.client.blogger.posts.insert({
        blogId: BLOG_ID,
        resource: {
          title: title,
          content: content,
          status: 'LIVE'
        }
      });
      statusElem.textContent = 'Post created successfully!';
      document.getElementById('postTitle').value = '';
      document.getElementById('postContent').value = '';
      listPosts();
    } catch (err) {
      statusElem.textContent = 'Error creating post: ' + err.message;
    }
  }

  // Load scripts
  window.onload = () => {
    gapiLoaded();

    const script = document.createElement('script');
    script.src = "https://accounts.google.com/gsi/client";
    script.onload = gisLoaded;
    document.body.appendChild(script);
  };
</script>

</body>
</html>
