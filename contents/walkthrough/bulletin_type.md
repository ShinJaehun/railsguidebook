# 게시판 레이아웃 작성하기

게시판의 형태는 흔히 세가지 정도로 분류할 수 있다.

* `일반형(bulletin)` : 일반적인 게시판의 형태. 디폴트 형태
* `블로그형(blog)` : 블그로와 게시물의 내용이 일렬로 보이도록 하는 형태
* `갤러리형(gallery)` : 이미지 갤러리처럼 한 줄에 여러개의 썸네일 이미지가 보이도록 하는 형태

### Bulletin 모델에 post_type 속성 추가하기

게시판을 새로 추가할 때, 레이아웃를 지정하기 위해서 `post_type`이라는 속성을 추가하기로 한다. 이 속성은 `string` 속성을 가지는 것으로 하고, `bulletin`, `blog`, `gallery` 값을 가질 수 있다. 이를 위해서 `Bulletin` 모델에 속성을 추가하는 마이그레이션 파일을 생성한다.

```bash
$ bin/rails g migration add_post_type_to_bulletins post_type
      invoke  active_record
      create    db/migrate/20140513063455_add_post_type_to_bulletins.rb
```

그리고 방금 전에 생성된 마이그레이션 파일을 열어 아래와 같이 `post_type`의 디폴트 값을 `bulletin`으로 추가하고,

```ruby
class AddPostTypeToBulletins < ActiveRecord::Migration
  def change
    add_column :bulletins, :post_type, :string, default: 'bulletin'
  end
end
```

저장한 후 `db:migrate` 작업을 한다.

```bash
$ bin/rake db:migrate
== 20140513063455 AddPostTypeToBulletins: migrating ===========================
-- add_column(:bulletins, :post_type, :string, {:default=>"bulletin"})
   -> 0.0037s
== 20140513063455 AddPostTypeToBulletins: migrated (0.0038s) ==================
```

### Strong Parameter 추가하기

`bulletins_controller.rb` 파일의 열어 하단에 있는 `bulletin_params` 메소드에 아래와 같이 `post_type` 속성을 추가한다.

```ruby
def bulletin_params
  params.require(:bulletin).permit(:title, :description, :post_type)
end
```

### Bulletin 뷰 템플릿 파일의 변경

`app/views/bulletins/_form.html.erb` 파일을 열어 아래의 코드를 추가해 준다.

```html
<div class="form-group">
  <%= f.input :post_type, collection: [ ['게시판', 'bulletin'], ['블로그', 'blog']], input_html:{ class:'form-control'} %>
</div>
```

![](http://i1373.photobucket.com/albums/ag392/rorlab/Photobucket%20Desktop%20-%20RORLAB/rcafe/2014-05-13_17-52-00_zps4b0eb3c6.png)

`app/views/bulletins/show.html.erb` 파일을 열어 아래의 코드를 추가해 준다.

```html
<tr>
  <th>Post Type</th>
  <td><%= @bulletin.post_type %></td>
</tr>
```

![](http://i1373.photobucket.com/albums/ag392/rorlab/Photobucket%20Desktop%20-%20RORLAB/rcafe/2014-05-13_17-55-07_zps869b50c5.png)

이렇게 해서 `Bulletin` 모델에서 추가할 작업이 완료되었다.

`http://localhost:3000/bulletins`로 접속해서 세 개의 `bulletin`을 등록한다.(`'공지사항'`, `'새소식'`, `'가입인사'`)

이제, 게시판의 형태에 따른 뷰를 보이도록 하기 위해서는 `app/views/posts/` 디렉토리에 `post_types`라는 하위 디렉토리를 만들고 이 디렉토리에 `_bulletin.html.erb` 파일과 `_blog.html.erb`, `_gallery_html.erb` 파일을 생성한다.

이제 `posts` 컨트롤러의 `index` 액션 뷰 파일의 모든 내용을 `_bulletin.html.erb` 파일로 옮기되, 마지막 `<%= link_to 'New Post', new_post_path, class: 'btn btn-default' %>` 부분을 `<%= link_to 'New Post', new_bulletin_post_path, class: 'btn btn-default' %>` 으로 수정하자.

```ruby
<%= render "posts/post_types/#{@bulletin.post_type}" %>
```

위에서 `render` 메소드는 `partial` 템플릿 파일을 인수로 받아 렌더링 결과를 삽입해 준다. 루비에서는 이중 인용부호 내의 `#{expression}`는 표현식의 결과로 대체해 준다. 따라서 `@bulletin.post_type` 값이 `'bulletin'`일 경우 `"posts/post_types/bulletin"`로 평가되어 `app/views/posts/post_types/` 디렉토리의 `_bulletin.html.erb`이라는 `partial` 템플릿 파일을 `render` 메소드가 처리하게 된다.

> **Note** `partial` 템플릿 파일에서는 부모 템플릿 파일에서 사용하는 모든 인스턴스 변수를 그대로 사용할 수 있다.

`_blog.html.erb` 파일의 내용을 아래와 같이 작성한다.

```html
<h2><%= params[:bulletin_id] %></h2>

<% @posts.each do | post | %>
    <div class='post'>
      <div class='title'><%= post.title %></div>
      <div class='content'><%= simple_format post.content %></div>
      <div>
          <%= link_to 'Show', [post.bulletin, post], class:'btn btn-default' %>
          <%= link_to 'Edit', edit_bulletin_post_path(post.bulletin, post), class:'btn btn-default' %>
          <%= link_to 'Destroy', [post.bulletin, post], method: :delete, data: { confirm: 'Are you sure?' }, class:'btn btn-default' %>
      </div>
    </div>
<% end %>

<br>

<%= link_to 'New Post', new_bulletin_post_path, class: 'btn btn-default' %>
```

그리고 `app/constrollers/posts_controller.rb` 파일을 열고 아래와 같이 변경한다.

```ruby
class PostsController < ApplicationController
  before_action :set_bulletin
  before_action :set_post, only: [:show, :edit, :update, :destroy]

  # GET /posts
  # GET /posts.json
  def index
    @posts = @bulletin.posts.all
  end

  # GET /posts/1
  # GET /posts/1.json
  def show
  end

  # GET /posts/new
  def new
    @post = @bulletin.posts.new
  end

  # GET /posts/1/edit
  def edit
  end

  # POST /posts
  # POST /posts.json
  def create
    @post = @bulletin.posts.new(post_params)

    respond_to do |format|
      if @post.save
        format.html { redirect_to [@post.bulletin, @post], notice: 'Post was successfully created.' }
        format.json { render :show, status: :created, location: @post }
      else
        format.html { render :new }
        format.json { render json: @post.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /posts/1
  # PATCH/PUT /posts/1.json
  def update
    respond_to do |format|
      if @post.update(post_params)
        format.html { redirect_to [@post.bulletin, @post], notice: 'Post was successfully updated.' }
        format.json { render :show, status: :ok, location: @post }
      else
        format.html { render :edit }
        format.json { render json: @post.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /posts/1
  # DELETE /posts/1.json
  def destroy
    @post.destroy
    respond_to do |format|
      format.html { redirect_to bulletin_posts_url, notice: 'Post was successfully destroyed.' }
      format.json { head :no_content }
    end
  end

  private
    def set_bulletin
      @bulletin = Bulletin.friendly.find(params[:bulletin_id])
    end

    def set_post
      @post = @bulletin.posts.find(params[:id])
    end

    def post_params
      params.require(:post).permit(:title, :content, :picture, :picture_cache)
    end
end
```

이제 `posts` 뷰 템플릿 파일들을 아래와 같이 수정한다.

### posts#new 뷰 템플릿 파일

```html
<h2><%= params[:bulletin_id] %></h2>

<%= render 'form' %>

<hr>
<%= link_to 'Back', bulletin_posts_path, class: 'btn btn-default' %>
```

### posts#edit 뷰 템플릿 파일

```html
<h2><%= params[:bulletin_id] %></h2>

<%= render 'form' %>

<hr>
<%= link_to 'Show', [@post.bulletin, @post], class: 'btn btn-default' %>
<%= link_to 'Back', bulletin_posts_path, class: 'btn btn-default' %>
```

### posts#show 뷰 템플릿 파일

```html
<h2><%= params[:bulletin_id] %></h2>

<table class='table'>
<tr>
  <th>Title</th>
  <td><%= @post.title %></td>
</tr>
<tr>
  <th>Content</th>
  <td><%= @post.content %></td>
</tr>
<tr>
  <th>Created at</th>
  <td><%= @post.created_at %></td>
</tr>
</table>


<%= link_to 'Edit', edit_bulletin_post_path(@post.bulletin, @post), class: 'btn btn-default' %>
<%= link_to 'Back', bulletin_posts_path, class: 'btn btn-default' %>
```

이제 브라우저에서 `http://localhost:3000/bulletins/가입인사/edit`로 접속한 후 게시판의 종류를 `블로그`로 변경한 후,

![](http://i1373.photobucket.com/albums/ag392/rorlab/Photobucket%20Desktop%20-%20RORLAB/rcafe/2014-05-13_18-24-22_zpsd9dcab9f.png)

브라우저에서 상단 메뉴 중 `가입인사`를 클릭해서 보면 아래와 같이 변경되어 보이게 된다.(가입인사를 테스트로 몇개 새로 추가한 후)

![](http://i1373.photobucket.com/albums/ag392/rorlab/Photobucket%20Desktop%20-%20RORLAB/rcafe/2014-05-13_18-27-05_zps11cfa325.png)

위와 같이 보이게 하기 위해서의 약간의 작업을 추가로 해 주어야 한다.
우선 `CSS` 클래스 `post`, `title`, `content`를 `app/assets/stylesheets/posts.css.scss` 파일에 작성해 준다.

```css
.post {
  border: 1px solid $gray-light;
  border-radius: 5px;
  padding:1em;
  margin-bottom: 1em;
  .title {
    font-weight: bold;
    font-size: 1.5em;
    margin-bottom:.5em;
  }
}
```

그리고, `app/assets/stylesheets/` 디렉토리에 있는 `application.css.scss` 파일을 열고 아래와 같이 작성한다.(`@import 'posts';`을 추가했음)

```html
$light-orange: #ff8c00;
$navbar-default-color: $light-orange;
$navbar-default-bg: #312312;
$navbar-default-link-color: gray;
$navbar-default-link-active-color: $light-orange;
$navbar-default-link-hover-color: white;
$navbar-default-link-hover-bg: black;

@import 'bootstrap';
@import 'posts';

body { padding-top: 60px; }
```


---
> **Git소스** https://github.com/rorlab/rcafe/tree/제5.11장
