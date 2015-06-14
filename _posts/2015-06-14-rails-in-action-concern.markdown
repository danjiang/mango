---
title: Rails 实战 - Concern
author: 但江
location: 成都
category: programming
---

Rails 项目中通常都是将业务逻辑写在 Model 中，有时候一些业务逻辑不仅仅只是在一个特定的 Model 中，有时候一些相同的业务逻辑适用于多个 Model，我们就可以利用 [ActiveSupport::Concern][2] 来做封装。

#### 重复的代码

在一个学校的管理系统中，一个学校有很多个相册，一个管理员只在一个学校，如何控制管理员在查看相册时候是它所属的学校的相册，还有很多如学校公告和学校食谱也属于这种情况，下面这种做法在很多地方都会重复：

{% highlight ruby %}
resources :school_albums do
    resources :school_album_images, :except => :show
end

def show
    @album = SchoolAlbum.find(params[:id])
    redirect_to root_path if @album.school_id != current_user.schools.first.id
    @images = @album.school_album_images.order("time DESC")
end
{% endhighlight %}

#### 不重复的代码

app/models/concerns/school_visible.rb 文件中

{% highlight ruby %}
module SchoolVisible
    extend ActiveSupport::Concern

    def visible_to(user)
        raise CanCan::AccessDenied if self.school_id != user.school.id
    end
end
{% endhighlight %}

app/models/school_ablum.rb 文件中

{% highlight ruby %}
class SchoolAlbum < ActiveRecord::Base
    self.table_name = "school_album"
    belongs_to :school, :foreign_key => "school_id"
    has_many :school_album_images, :foreign_key => "album_id", :dependent => :destroy

    validates :title, :presence => true, :length => { :maximum => 14 }
    include SchoolVisible
end
{% endhighlight %}

在 Controller 中的代码变得更直观和简洁：

{% highlight ruby %}
def show
    @album = SchoolAlbum.find(params[:id])
    @album.visible_to(current_user)
    @images = @album.school_album_images.order("time DESC")
end
{% endhighlight %}

#### 参考

* [Put chubby models on a diet with concerns][1]
* [ActiveSupport::Concern][2]

[1]: https://signalvnoise.com/posts/3372-put-chubby-models-on-a-diet-with-concerns
[2]: http://api.rubyonrails.org/classes/ActiveSupport/Concern.html 
