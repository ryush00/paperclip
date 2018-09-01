# Paperclip에서 ActiveStorage로 이전하기

Paperclip과 ActiveStorage는 비슷한 문제를 비슷한 해결책을 통해서 해결하므로, 이동은 간단한 데이터 재작성을 통해 이뤄질수 있습니다.

Papercip에서 ActiveStorage로의 이전 과정은 아래와 같습니다:

1. ActiveStorage 데이터베이스 마이그레이션 적용하기
2. 저장소 설정하기
3. 데이터베이스 데이터 복사하기
4. 파일 복사하기
5. 테스트 수정하기
6. 뷰 수정하기
7. 컨트롤러 수정하기
8. 모델 수정하기

## ActiveStorage 데이터베이스 마이그레이션 적용하기

[ActiveStorage 설치 가이드]를 완료하세요. `mini_magick` gem을 Gemfile에 추가할것입니다.

```sh
rails active_storage:install
```

[ActiveStorage 설치 가이드]: https://github.com/rails/rails/blob/master/activestorage/README.md#installation

## 저장소 설정하기

이제, [ActiveStorage 설정 가이드]를 완료하세요.

[ActiveStorage 설정 가이드]: http://edgeguides.rubyonrails.org/active_storage_overview.html#setup

## 데이터베이스 데이터 복사하기

`active_storage_blobs` 및  `active_storage_attachments` 테이블은 ActiveStorage가 파일 메타데이터를 찾는 곳입니다. Paperclip은 메타데이터를 관련된 객체의 테이블에 직접 저장합니다.

You'll need to write a migration for this conversion. Because the models for
your domain are involved, it's tricky to supply a simple script. But we'll try!

Here's how it would go for a `User` with an `avatar`, that is this in
Paperclip:

```ruby
class User < ApplicationRecord
  has_attached_file :avatar
end
```

Papercip 마이그레이션은 아래와 같은 테이블을 생성할 것입니다:

```ruby
create_table "users", force: :cascade do |t|
  t.string "avatar_file_name"
  t.string "avatar_content_type"
  t.integer "avatar_file_size"
  t.datetime "avatar_updated_at"
end
```

And you'll be converting into these tables:

```ruby
create_table "active_storage_attachments", force: :cascade do |t|
  t.string "name", null: false
  t.string "record_type", null: false
  t.integer "record_id", null: false
  t.integer "blob_id", null: false
  t.datetime "created_at", null: false
  t.index ["blob_id"], name: "index_active_storage_attachments_on_blob_id"
  t.index ["record_type", "record_id", "name", "blob_id"], name: "index_active_storage_attachments_uniqueness", unique: true
end
```

```ruby
create_table "active_storage_blobs", force: :cascade do |t|
  t.string "key", null: false
  t.string "filename", null: false
  t.string "content_type"
  t.text "metadata"
  t.bigint "byte_size", null: false
  t.string "checksum", null: false
  t.datetime "created_at", null: false
  t.index ["key"], name: "index_active_storage_blobs_on_key", unique: true
end
```

So, assuming you want to leave the files in the exact same place,  _this is
your migration_. Otherwise, see the next section first and modify the migration
to taste.

```ruby
Dir[Rails.root.join("app/models/**/*.rb")].sort.each { |file| require file }

class ConvertToActiveStorage < ActiveRecord::Migration[5.2]
  require 'open-uri'

  def up
    # postgres
    get_blob_id = 'LASTVAL()'
    # mariadb
    # get_blob_id = 'LAST_INSERT_ID()'
    # sqlite
    # get_blob_id = 'LAST_INSERT_ROWID()'

    active_storage_blob_statement = ActiveRecord::Base.connection.raw_connection.prepare(<<-SQL)
      INSERT INTO active_storage_blobs (
        `key`, filename, content_type, metadata, byte_size, checksum, created_at
      ) VALUES (?, ?, ?, '{}', ?, ?, ?)
    SQL

    active_storage_attachment_statement = ActiveRecord::Base.connection.raw_connection.prepare(<<-SQL)
      INSERT INTO active_storage_attachments (
        name, record_type, record_id, blob_id, created_at
      ) VALUES (?, ?, ?, #{get_blob_id}, ?)
    SQL

    models = ActiveRecord::Base.descendants.reject(&:abstract_class?)

    transaction do
      models.each do |model|
        attachments = model.column_names.map do |c|
          if c =~ /(.+)_file_name$/
            $1
          end
        end.compact

        model.find_each.each do |instance|
          attachments.each do |attachment|
            active_storage_blob_statement.execute(
              key(instance, attachment),
              instance.send("#{attachment}_file_name"),
              instance.send("#{attachment}_content_type"),
              instance.send("#{attachment}_file_size"),
              checksum(instance.send(attachment)),
              instance.updated_at.iso8601
            )

            active_storage_attachment_statement.
              execute(attachment, model.name, instance.id, instance.updated_at.iso8601)
          end
        end
      end
    end

    active_storage_attachment_statement.close
    active_storage_blob_statement.close
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end

  private

  def key(instance, attachment)
    SecureRandom.uuid
    # Alternatively:
    # instance.send("#{attachment}_file_name")
  end

  def checksum(attachment)
    # local files stored on disk:
    url = attachment.path
    Digest::MD5.base64digest(File.read(url))

    # remote files stored on another person's computer:
    # url = attachment.url
    # Digest::MD5.base64digest(Net::HTTP.get(URI(url)))
  end
end
```

## 파일 복사하기

위 마이그레이션은 파일을 그대로 놔둡니다. 그러나, 기본적으로 Paperclip과 ActiveStorage 저장소는 다른 위치에 파일을 저장합니다.

기본적으로 Paperclip은 이런 방식으로 파일을 저장하고:

```
public/system/users/avatars/000/000/004/original/the-mystery-of-life.png
```

ActiveStorage는 이런 방식으로 파일을 저장합니다:

```
storage/xM/RX/xMRXuT6nqpoiConJFQJFt6c9
```

저 `xMRXuT6nqpoiConJFQJFt6c9` 값은 `active_storage_blobs.key` 값입니다. 윗 마이그레이션에서 우리는 간단히 파일명을 사용했지만, UUID를 사용하고 싶을수도 니다.

### Moving local storage files

```ruby
#!bin/rails runner

class ActiveStorageBlob < ActiveRecord::Base
end

class ActiveStorageAttachment < ActiveRecord::Base
  belongs_to :blob, class_name: 'ActiveStorageBlob'
  belongs_to :record, polymorphic: true
end

ActiveStorageAttachment.find_each do |attachment|
  name = attachment.name

  source = attachment.record.send(name).path
  dest_dir = File.join(
    "storage",
    attachment.blob.key.first(2),
    attachment.blob.key.first(4).last(2))
  dest = File.join(dest_dir, attachment.blob.key)

  FileUtils.mkdir_p(dest_dir)
  puts "Moving #{source} to #{dest}"
  FileUtils.cp(source, dest)
end
```

### Moving files on a remote host (S3, Azure Storage, GCS, etc.)

One of the most straightforward ways to move assets stored on a remote host is
to use a rake task that regenerates the file names and places them in the
proper file structure/hierarchy.

Assuming you have a model configured similarly to the example below:

```ruby
class Organization < ApplicationRecord
  # New ActiveStorage declaration
  has_one_attached :logo

  # Old Paperclip config
  # must be removed BEFORE to running the rake task so that
  # all of the new ActiveStorage goodness can be used when
  # calling organization.logo
  has_attached_file :logo,
                    path: "/organizations/:id/:basename_:style.:extension",
                    default_url: "https://s3.amazonaws.com/xxxxx/organizations/missing_:style.jpg",
                    default_style: :normal,
                    styles: { thumb: "64x64#", normal: "400x400>" },
                    convert_options: { thumb: "-quality 100 -strip", normal: "-quality 75 -strip" }
end
```

The following rake task would migrate all of your assets:

```ruby
namespace :organizations do
  task migrate_to_active_storage: :environment do
    Organization.where.not(logo_file_name: nil).find_each do |organization|
      # This step helps us catch any attachments we might have uploaded that
      # don't have an explicit file extension in the filename
      image = organization.logo_file_name
      ext = File.extname(image)
      image_original = URI.unescape(image.gsub(ext, "_original#{ext}"))

      # this url pattern can be changed to reflect whatever service you use
      logo_url = "https://s3.amazonaws.com/xxxxx/organizations/#{organization.id}/#{image_original}"
      organization.logo.attach(io: open(logo_url),
                                   filename: organization.logo_file_name,
                                   content_type: organization.logo_content_type)
    end
  end
end
```

An added advantage of this method is that you're creating a copy of all assets,
which is handy in the event you need to rollback your deploy.

This also means that you can run the rake task from your development machine
and completely migrate the assets before your deploy, minimizing the chances
that you'll have a timed-out deployment.

The main drawback of this method is the same as its benefit - you are
essentially duplicating all of your assets. These days storage and bandwidth
are relatively cheap, but in some instances where you have a huge volume of
files, or very large file sizes, this might get a little less feasible.

In my experience I was able to move tens of thousands of images in a matter of
a couple of hours, just by running the migration overnight on my MacBook Pro.

Once you've confirmed that the migration and deploy have gone successfully you
can safely delete the old assets from your remote host.

## Update your tests

Instead of the `have_attached_file` matcher, you'll need to write your own.
Here's one that is similar in spirit to the Paperclip-supplied matcher:

```ruby
RSpec::Matchers.define :have_attached_file do |name|
  matches do |record|
    file = record.send(name)
    file.respond_to?(:variant) && file.respond_to?(:attach)
  end
end
```

## Update your views

In Paperclip it looks like this:

```ruby
image_tag @user.avatar.url(:medium)
```

In ActiveStorage it looks like this:

```ruby
image_tag @user.avatar.variant(resize: "250x250")
```

## Update your controllers

This should _require_ no update. However, if you glance back at the database
schema above, you may notice a join.

For example, if your controller has

```ruby
def index
  @users = User.all.order(:name)
end
```

And your view has

```
<ul>
  <% @users.each do |user| %>
    <li><%= image_tag user.avatar.variant(resize: "10x10"), alt: user.name %></li>
  <% end %>
</ul>
```

Then you'll end up with an n+1 as you load each attachment in the loop.

So while the controller and model will work without change, you will want to
double-check your loops and add `includes` as needed. ActiveStorage adds an
`avatar_attachment` and `avatar_blob` relationship to has-one relations, and
`avatar_attachments` and `avatar_blobs` to has-many:

```ruby
def index
  @users = User.all.order(:name).includes(:avatar_attachment)
end
```

## Update your models

Follow [the guide on attaching files to records]. For example, a `User` with an
`avatar` is represented as:

```ruby
class User < ApplicationRecord
  has_one_attached :avatar
end
```

Any resizing is done in the view as a variant.

[the guide on attaching files to records]: http://edgeguides.rubyonrails.org/active_storage_overview.html#attaching-files-to-records

## Paperclip 제거하기

Gem을 `Gemfile`에서 제거하고 `bundle` 명령어를 실행하세요. 그리고 테스트를 실행하면 끝!
