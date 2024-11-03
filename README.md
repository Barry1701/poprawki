# DRESTF-API

This project is a Django REST framework (DRF) API designed to provide backend functionality for a social media platform that supports user interactions through comments, likes, follows, and posts. The API also includes features for managing user profiles and products, offering endpoints for various actions, including authentication, CRUD operations for posts, and social interactions between users.

## Project Structure

The repository structure is as follows:

- **comments/** - Contains functionality for managing comments on posts. This app handles endpoints for creating, viewing, editing, and deleting comments.
  
- **drf_api/** - Likely the core app of the project, possibly containing project settings, configurations, and any custom configurations for the Django REST framework.
  
- **followers/** - Manages follower relationships between users, enabling users to follow and unfollow others.
  
- **likes/** - Provides functionality for users to like posts, supporting endpoints to like and unlike specific posts.
  
- **posts/** - Manages the creation, retrieval, updating, and deletion of posts. This is a core app that provides social content on the platform.
  
- **products/** - Likely handles data related to products. This could include CRUD operations for product listings, if relevant to the app’s purpose.
  
- **profiles/** - Manages user profile data, allowing users to view, update, and manage personal information.

## Additional Files

- **.gitignore** - Specifies files and directories that Git should ignore (e.g., environment files, compiled code).
  
- **db.sqlite3** - The SQLite database file for storing data in a local development environment.
  
- **env.py** - Contains environment variables, such as sensitive credentials or configuration variables, required for local development.
  
- **manage.py** - A command-line utility that lets you interact with this Django project in various ways (e.g., running the server, migrating the database).
  
- **Procfile** - Used to declare the commands to run the app on a hosting platform (such as Heroku).
  
- **README.md** - The README file, where this documentation will be expanded to provide a full overview of the project.
  
- **requirements.txt** - Lists all the Python dependencies that need to be installed for this project, which ensures consistent environment setup.

## Comments App

The `comments` app in this project handles user comments on posts. It includes models, serializers, views, and URL configurations to support CRUD operations for comments, allowing users to create, view, edit, and delete comments.

### Models

The `Comment` model represents a comment that a user can post on a specific post. It includes fields to associate each comment with a user and a post, as well as timestamp fields to track when the comment was created and last updated.

```python
from django.db import models 
from django.contrib.auth.models import User
from posts.models import Post

class Comment(models.Model):
    """
    Comment model, related to User and Post
    """
    owner = models.ForeignKey(User, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    content = models.TextField()

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.content
```

- **Fields**:
  - `owner`: Foreign key to the `User` model, representing the user who owns the comment.
  - `post`: Foreign key to the `Post` model, representing the post that the comment belongs to.
  - `created_at`: Timestamp for when the comment was created.
  - `updated_at`: Timestamp for when the comment was last updated.
  - `content`: Text content of the comment.

- **Meta Options**:
  - `ordering`: Comments are ordered by `created_at` in descending order.

- **String Representation**:
  - Returns the content of the comment as its string representation.
 
## Serializers

The serializers convert `Comment` model instances into JSON and add additional fields for the API response.

### `CommentSerializer`

Adds fields to represent additional data related to the owner (username, profile image) and timestamps in human-readable format.

```python
from django.contrib.humanize.templatetags.humanize import naturaltime
from rest_framework import serializers
from .models import Comment

class CommentSerializer(serializers.ModelSerializer):
    """
    Serializer for the Comment model
    Adds three extra fields when returning a list of Comment instances
    """
    owner = serializers.ReadOnlyField(source='owner.username')
    is_owner = serializers.SerializerMethodField()
    profile_id = serializers.ReadOnlyField(source='owner.profile.id')
    profile_image = serializers.ReadOnlyField(source='owner.profile.image.url')
    created_at = serializers.SerializerMethodField()
    updated_at = serializers.SerializerMethodField()

    def get_is_owner(self, obj):
        request = self.context['request']
        return request.user == obj.owner

    def get_created_at(self, obj):
        return naturaltime(obj.created_at)

    def get_updated_at(self, obj):
        return naturaltime(obj.updated_at)

    class Meta:
        model = Comment
        fields = [
            'id', 'owner', 'is_owner', 'profile_id', 'profile_image',
            'post', 'created_at', 'updated_at', 'content'
        ]

```

- **Fields**:

  - `owner`: Read-only field that displays the username of the comment owner.
  - `is_owner`: Boolean field indicating if the requesting user is the owner of the comment.
  - `profile_id`: Read-only field that displays the profile ID of the comment owner.
  - `profile_image`: Read-only field that displays the URL of the profile image of the comment owner.
  - `created_at`: Human-readable timestamp for when the comment was created.
  - `updated_at`: Human-readable timestamp for when the comment was last updated.

---

- **Methods**:

  - `get_is_owner`: Determines if the current user is the owner of the comment.
  - `get_created_at`: Converts the `created_at` timestamp to a human-readable format.
  - `get_updated_at`: Converts the `updated_at` timestamp to a human-readable format.

---

### `CommentDetailSerializer`

Used for detailed views of comments, making the `post` field read-only.

```python
class CommentDetailSerializer(CommentSerializer):
    """
    Serializer for the Comment model used in Detail view
    Post is a read-only field so that we don’t have to set it on each update
    """
    post = serializers.ReadOnlyField(source='post.id')

```

### Fields

- **`post`**: Read-only field displaying the ID of the associated post. This allows for viewing the post ID without the need to set it on each update.

## Views

The views provide the logic for listing, creating, retrieving, updating, and deleting comments.

### `CommentList`

Handles listing all comments or creating a new comment if the user is authenticated. Supports filtering by `post` to retrieve comments specific to a post.

```python
from rest_framework import generics, permissions
from django_filters.rest_framework import DjangoFilterBackend
from drf_api.permissions import IsOwnerOrReadOnly
from .models import Comment
from .serializers import CommentSerializer, CommentDetailSerializer

class CommentList(generics.ListCreateAPIView):
    """
    List comments or create a comment if logged in.
    """
    serializer_class = CommentSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    queryset = Comment.objects.all()
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['post']

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

```

### Methods

- **`perform_create`**: Sets the `owner` of the comment to the current user when creating a new comment.

---

### Attributes

- **`serializer_class`**: Specifies the serializer to use for the `Comment` model.
- **`permission_classes`**: Allows authenticated users to create comments; others can only view.
- **`queryset`**: Defines the base queryset to retrieve all comments.
- **`filter_backends`**: Enables filtering by `post` ID.
- **`filterset_fields`**: Allows filtering of comments by the `post` field.

### `CommentDetail`

Provides a detail view, allowing the user to retrieve, update, or delete a comment by ID if they own it.

```python
from rest_framework import generics
from drf_api.permissions import IsOwnerOrReadOnly
from .models import Comment
from .serializers import CommentDetailSerializer

class CommentDetail(generics.RetrieveUpdateDestroyAPIView):
    """
    Retrieve a comment, or update or delete it by id if you own it.
    """
    permission_classes = [IsOwnerOrReadOnly]
    serializer_class = CommentDetailSerializer
    queryset = Comment.objects.all()

```

### Attributes

- **`permission_classes`**: Restricts update and delete actions to the comment owner; others have read-only access.
- **`serializer_class`**: Specifies the `CommentDetailSerializer` for detailed representation.
- **`queryset`**: Defines the base queryset to retrieve comments.

## URLs

**`urls.py`**

Defines the URL patterns for accessing comment-related views.

```python
from django.urls import path
from comments import views

urlpatterns = [
    path('comments/', views.CommentList.as_view()),
    path('comments/<int:pk>/', views.CommentDetail.as_view())
]

```

- **`comments/`**: URL for listing all comments or creating a new comment.
- **`comments/<int:pk>/`**: URL for retrieving, updating, or deleting a specific comment by its primary key (ID).
























