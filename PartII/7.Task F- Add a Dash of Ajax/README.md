# Task F: Add a Dash of Ajax

- 用 partial
- 將 parital render 成頁面
- 用 Ajax 和 Javascript 來動態更新頁面
- 用 jQuery UI 來提示有變更的地方
- 隱藏或顯示 DOM 元素
- 用 WebSockets 來廣播有變更的地方
- 對 Ajax updates 做測試

### 什麼是 Ajax?

Ajax lets us write code that runs in the browser that interacts with our server- based application. In our case, we’d like to make the Add to Cart buttons invoke the server create action on the LineItems controller in the background. The server can then send down just the HTML for the cart, and we can replace the cart in the sidebar with the server’s updates.

### 什麼是 DOM?

### 什麼是 Partial?


### 什麼是 Helper?
Whenever we want to abstract some processing out of a view (any kind of view), we should write a helper method. The Rails generators automatically created a helper file for each of our con- trollers. The Rails command itself (the one that created the application initially) created the file application_helper.rb. If you like, you can organize your methods into controller-specific helpers, but because this method will be used in the application layout, let’s put it in the application helper.

### 什麼是 CoffeeScript?


## Iteration F1: Moving the Cart

原本的 rails50/depot_j/app/views/carts/show.html.erb
```erb
<p id="notice"><%= notice %></p>
<h2>Your Cart</h2>
<table>
  <%= render(@cart.line_items) %>
  <tr class="total_line">
    <td colspan="2">Total</td>
    <td class="total_cell"><%= number_to_currency(@cart.total_price) %></td>
  </tr>
</table>
<%= button_to 'Empty cart', @cart, method: :delete, data: { confirm: 'Are you sure?' } %>
```

生出以下兩個 partials:

rails50/depot_j/app/views/line_items/_line_item.html.erb
```erb
<tr>
  <td><%= line_item.quantity %>&times;</td>
  <td><%= line_item.product.title %></td>
  <td class="item_price"><%= number_to_currency(line_item.total_price) %></td>
</tr>
```

rails50/depot_j/app/views/carts/_cart.html.erb
```erb
<p id="notice"><%= notice %></p>
<h2>Your Cart</h2>
<table>
  <%= render(cart.line_items) %>
  <tr class="total_line">
    <td colspan="2">Total</td>
    <td class="total_cell"><%= number_to_currency(cart.total_price) %></td>
  </tr>
</table>
<%= button_to 'Empty cart', cart, method: :delete, data: { confirm: 'Are you sure?' } %>
```

第二個 partial 中的 `<%= render(cart.line_items) %>` 是在 render 第一個 partial

現在 rails50/depot_k/app/views/carts/show.html.erb 變成下面這樣：
```erb
<p id="notice"><%= notice %></p>
<%= render @cart %>
```

---

把這個新的 partial 裝進 application layout 的側欄

rails50/depot_k/app/views/layouts/application.html.erb
```erb
  ......

  <body class="<%= controller.controller_name %>">
    <div id="banner">
      <%= image_tag("logo.png") %>
      <span class="title"><%= @page_title || "Pragmatic Bookshelf" %></span>
    </div>
    <div id="columns">
      <div id="side"> 
+       <div id="cart">
+         <%= render @cart %> 
+       </div>

  ......
```

我們跑 store controller index action 時也會使用上面這個 layout, 但這時 index action 還沒有設定 @cart, 所以我們來做個小修改：

rails50/depot_k/app/controllers/store_controller.rb
```rb
  class StoreController < ApplicationController 
+   include CurrentCart
+   before_action :set_cart
    def index
      @products = Product.order(:title)
    end  
  end
```

接著，處理前端視覺效果。

最後，我們要讓使用者按下 "Add to Cart" 按鈕後，側欄重刷 index 頁面來更新購物車內容。方法是在 line_items_controller.rb 中的 create action 最後加上一行 `format.html { redirect_to store_url }`。

Iteration F1 到此結束，現在側欄有一台購物車了。

---

## Iteration F2: Creating an Ajax-Based Cart

上個小節的購物車有一個要改進的地方。每次我們按下 "Add to Cart" 按鈕一次，就會重刷一次頁面，吃資源也影響效能，因此，我們要來建一台 Ajax-based cart.

一般的做法是，在伺服端寫代碼接客戶端寫的 JavaScript(通常是JSON)，但在 Rails 中，我們可以直接寫 ruby 以及使用一大堆 Rails helper 來做到一樣的事。

將程式 Ajax 化的訣竅是一小步一小步進行：

第一步：改 catalog 頁面，讓它能發出 Ajax request 到伺服端，伺服端再傳回要更動的特定 HTML 片段。

在 rails50/depot_l/app/views/store/index.html.erb 中，我們在 "Add to Cart" 那段代碼後面加上 `remote: true`，變成 `<%= button_to 'Add to Cart', line_items_path(product_id: product),
remote: true %>`，這樣就能讓 "Add to Cart" 按鈕發出 Ajax request.

第二步：讓伺服端送出 response

方法是產生更新過的購物車的 HTML 片段，讓瀏覽器的 DOM 去接這段 HTML. 我們利用 DOM 來決定使用者在前端看到什麼。

這邊，我們會設個條件，如果 request 的內容是 JavaScript,就讓 create action 不會 redirect_to index.
方法是在 create action 中的 respond_to() 內加上 `format.js`，這樣當 create action 發出 Ajax request, Rails 就會去找 create template（模板）來 render。Rails 支援能產生 JavaScript 的 template, 比如 `.js.erb`，我們可以直接寫伺服端的 Ruby code 在 `.js.erb` 中來跑 JavaScript.

雖然 Rails 把 Ajax 實作變得很簡單，但並無法完全避免出錯，而 Ajax 的實作實際上是很多技術整合在一起的結果，假如 Ajax 無法運作，很難輕易判斷問題出在哪，因此一小步一小步進行，是比較適合的作法。

---

## Iteration F3: Highlighting Changes

由於 Ajax 更動的畫面實在太細微，使用者未必會注意到，這樣造成"我剛剛按下'Add to Cart'按鈕，程式到底有沒有動？"之類的困擾。因此，我們用 jQuery UI 來做提示更動的功能。

先進行以下設定：
Gemfile `gem "jquery-ui-rails"`
application.js `//= require jquery-ui/effect-blind`

接著要找出 cart 中的哪個 item 更新了，才能針對這個 item 做特效。

首先，設定 `@current_item ＝ @line_item`
line_items_controller.rb
```rb
  def create
    product = Product.find(params[:product_id]) 
    @line_item = @cart.add_product(product.id)
    respond_to do |format| 
      if @line_item.save
        format.html { redirect_to store_url }
+       format.js { @current_item = @line_item }
        format.json { render :show, status: :created, location: @line_item }
      else
        format.html { render :new }
        format.json { render json: @line_item.errors, status: :unprocessable_entity } 
      end
    end 
  end
```

接著，在 `_line_item.html.erb` partial 中，檢查 render 的 item 是不是剛剛更新的那個 item. 
views/line_items/_line_item.html.erb
```erb
+ <% if line_item == @current_item %>
+   <tr id="current_item">
+ <% else %>
+   <tr>
+ <% end %>
    <td><%= line_item.quantity %>&times;</td>
    <td><%= line_item.product.title %></td>
    <td class="item_price"><%= number_to_currency(line_item.total_price) %></td>
  </tr>
```

上面兩個改變讓"cart 中最後更新過的 item" 這個 `<tr>` 元素 `id="current_item"`，接著在 `views/line_items/create.js.erb` 中告訴 JavaScript 我們想要怎樣突顯有更動的地方：

方法是在 `$` function 中代入 `"#current_item"`，後面接 `.css()` 設定特效的起始背景色，再接 `.animate()` 設定特效的最終背景色，也就是我們的頁面底色；最後的 `1000` 則代表整個特效用 1000 毫秒、也就是 1 秒來完成。

views/line_items/create.js.erb
```erb
  $('#cart').html("<%= escape_javascript render(@cart) %>");

+ $('#current_item').css({'background-color':'#88ff88'}).
+ animate({'background-color':'#114411'}, 1000);
```

Iteration F3 到此結束，現在按下 "Add to Cart" 按鈕，有更新的地方就會有"閃一下"的特效了。

---

## Iteration F4: Hiding an Empty Cart

這個小節要把空的 cart 藏起來。

實作方法很多，最簡單的一個是在 _cart partial 中寫條件式來藏 cart, 但是這樣會造成 Ajax 更新時整個側欄都會"閃一下"，所以教材換第二個方法 - jQuery UI.

首先，我們在 `create.js.erb` 中透過 show('blind') 讓 cart 顯現出來：

```erb
+ if ($('#cart tr').length == 1) { $('#cart').show('blind', 1000); }
  $('#cart').html("<%= escape_javascript render(@cart) %>");   
  $('#current_item').css({'background-color':'#88ff88'}).
    animate({'background-color':'#114411'}, 1000);
```

為什麼這邊要用 show() 呢？因為空的 cart 會被隱藏，假如我們要觸發"閃一下"的特效，必須先讓 cart 在側欄 show 出來，"閃一下"的特效才有標的物去作用。

接著，我們要把空的 cart 藏起來，有一好一壞兩種簡單的方法。壞方法是最一開始就不要產生任何 HTML, 但這樣做的話，當我們第一次把商品加入 cart 時，HTML 片段突然產生，這時 cart 會先突然顯現出來，接著才跑"閃一下"的特效，視覺上的先後順序會怪怪的。

比較好的做法是最一開始仍然產生 cart 的 HTML，但假如 cart 是空的，就用 CSS 藏起來。方法如下：

app/views/layouts/application.html.erb
```erb
<div id="cart"
    <% if @cart.line_items.empty? %>
        style="display: none"
    <% end %>
  >
  <%= render(@cart) %> 
</div>
```

這個寫法很醜，比如說 line 5 的 `>` 看起來像寫錯了(實際上沒有)，或者在 `<div>` tag 中寫進邏輯......等。改善的方法是，我們先在 application_helper.rb 中寫一個叫 `hidden_div_if()` 的 helper:

helpers/application_helper.rb
```rb
  module ApplicationHelper
+   def hidden_div_if(condition, attributes = {}, &block)
+     if condition
+       attributes["style"] = "display: none"
+     end
+     content_tag("div", attributes, &block) 
+   end
  end
```

接著在 application.html.erb 中使用這個 helper:

views/layouts/application.html.erb
```erb
<%= hidden_div_if(@cart.line_items.empty?, id: 'cart') do %> 
  <%= render @cart %>
<% end %>
```

最後，我們要把"已清空購物車"的 message 從 flash 拿掉，一方面是 cart 被清空的時候，側欄的 cart 會(因為Ajax而)自動消失；另一方面，我們現在是用 Ajax 把商品加入 cart 並刷新側欄 cart, 而不是刷新主頁面，如果不拿掉 flash message, 即使側欄因為加入商品而顯現 cart, 也就是這時 cart 不是被清空的狀態，但主頁面在前一個動作產生的"已清空購物車" message 仍然不會消失。

Iteration F4 結束，我們打造出 Ajax-Based Cart 了。

## Iteration F5: Broadcasting Updates

前面重點放在使用者產生更動時，頁面的視覺響應上；這個小節則要處理其他人造成更動時，頁面的視覺響應，比如說"實時更新"的功能。Ajax 做不到這件事，但好消息是， Rails 5 的新功能 Action Cable 可以輕易做到。

使用 Action Cable 總共三個步驟：
1. 建立一個 channel 
2. broadcast 一些資料
3. 接收資料

### 1. 建立一個 channel:

新增 `app/channels/application_cable/products_channel.rb`
```rb
class ProductsChannel < ActionCable::Channel::Base 
  def subscribed
    stream_from "products" 
  end
end
```

兩個地方要注意：
1. name of the class `ProductsChannel`
2. name of the stream `products`

一個 channel 能支援多個 streams, 比如說聊天 app 可以有好多間聊天室，但我們目前只用到一個 stream.

Channels 有潛在的安全問題，所以在 development 模式下，Rails 預設只允許從 localhost 來運作。如果你需要在多台機器上開發，那就關掉這個設定，做法是：

config/environments/development.rb
```rb
+ config.action_cable.disable_request_forgery_protection = true
```

### 2. broadcast 一些資料

這個步驟的目標是頁面一有更動，就會 broadcast 整個 catalog。 雖然我們可以選擇只送出 catalog 的一部分或者任何其他資料，但既然我們已經有 catalog 的 view 了，那就直接用它吧。

app/controllers/products_controller.rb
```rb
  def update
    respond_to do |format|
      if @product.update(product_params) 
        format.html { redirect_to @product,
          notice: 'Product was successfully updated.' }
        format.json { render :show, status: :ok, location: @product }
        
+       @products = Product.all 
+       ActionCable.server.broadcast 'products',
+         html: render_to_string('store/index', layout: false) 
      else
        format.html { render :edit }
        format.json { render json: @product.errors,
          status: :unprocessable_entity } 
      end
    end 
  end
```

第一行：因為我們要用 store/index view, 所以先 `@products = Product.all`。
第二行：Broadcast "products" stream 的寫法。
第三行：我們用 render_to_string() 來把 "store/index" 當成 string 來 render, `layout: false` 是指我們只想要 "store/index" 這個 view 而不是整個頁面。Broadcasts messages 通常是以 Ruby hashes 為元素，轉成 JSON 加以利用，最後成為 JavaScript 物件。第三行的 `html` 被我們當作 hash key.

貼上下方網路上找到的一組範例當參考：
```
class WebNotificationsChannel < ApplicationCable::Channel
   def subscribed
     stream_from "web_notifications_#{current_user.id}"
   end
 end

 # Somewhere in your app this is called, perhaps from a NewCommentJob
 ActionCable.server.broadcast \
   "web_notifications_1", { title: 'New things!', body: 'All shit fit for print' }

 # Client-side coffescript which assumes you've already requested the right to send web notifications
 App.cable.subscriptions.create "WebNotificationsChannel",
   received: (data) ->
     new Notification data['title'], body: data['body']
```

### 3. 接收資料

最後一步是接收資料，包括：
1. 訂閱哪個 channel 
2. 定義收到資料後要做什麼

app/assets/javascripts/products.coffee
```coffee
   # Place all the behaviors and hooks related to the matching controller here.
   # All this logic will automatically be available in application.js.
   # You can use CoffeeScript in this file: http://coffeescript.org/
+  App.productsChannel =
+    App.cable.subscriptions.create { channel: "ProductsChannel" },  
+      received: (data) -> $(".store #main").html(data.html)
```

我們用 CoffeeScript 處理這部分，讓我們能用更簡明的形式來表達 JavaScript, 再結合 jQuery 就能輕鬆做出驚人效果。

這三行的作用是，新增一個 ProductsChannel 的 subscription 以及定義一個功能，當這個 channel 收到資料時就會執行這個功能。

這個功能會去找 帶著 main ID 的 HTML 有沒有包含在另一個 Store class 中；如果有找到，從 channel 接收到的資料就會把這段 HTML 代換掉，而整個頁面的其他部分並不會受影響。

上面這些都可以用 JavaScript 做到，但是會費事許多。做完記得重開 server. 


## Testing Ajax Changes

ActionCable 的功能做完了，接下來做測試：

`$ rails test` (在 Rails 4 是 `$ rake test`)

會出現下面這個 error:

`ActionView::Template::Error: undefined method `line_items' for nil:NilClass`


從 error 中判斷要用下面的方式解決：
rails50/depot_o/app/views/layouts/application.html.erb
```erb
+ <% if @cart %>
    <%= hidden_div_if(@cart.line_items.empty?, id: 'cart') do %>
      <%= render @cart %> 
    <% end %>
+ <% end %>
```

解決完再跑一次測試，這次只剩一個 error: Redirect 的值不合預期。

這個問題發生在新增一個 line item 的時候，而這是這本書在 Iteration F1 "Changing the Flow" (page 153) 故意留下的爛攤子，我們調整測試方式：

test/controllers/line_items_controller_test.rb
```rb
  test "should create line_item" do
    assert_difference('LineItem.count') do
      post line_items_url, params: { product_id: products(:ruby).id }
    end
    follow_redirect!
+   assert_select 'h2', 'Your Cart'
+   assert_select 'td', "Programming Ruby 1.9"
  end
```

問題解決了，而這段刻意安排的錯誤要表達的是，為了新功能而做的更動有可能造成前面做的功能出問題。哪怕是像 Depot 這樣的小應用，如果不小心，就會發生這類的錯誤；而如果是大的應用，再怎麼小心還是可能發生。

我們的測試還沒結束，接下來測試 Ajax - 也就是我們按下 "Add to Cart" 後的部分：

新增 test/controllers/line_items_controller_test.rb
```rb
test "should create line_item via ajax" do 
  assert_difference('LineItem.count') do
    post line_items_url, params: { product_id: products(:ruby).id }, 
      xhr: true
  end
  assert_response :success 
  assert_select_jquery :html, '#cart' do
    assert_select 'tr#current_item td', /Programming Ruby 1.9/ 
  end
end
```

這個測試的目標：
- 能夠成功代換一段新的 cart 的 HTML
- 這段新的 HTML 要有包含 current_item ID 的 row
- 這個 row 的值要 match "Programming Ruby 1.9"

透過 `assert_select_jquery()` 來取出相關的 HTML, 然後透過任何其他的 assertions 來處理這段 HTML, 我們就能達到這個測試的目標。

關於 assertion，可以參考下面兩個連結：
http://stackoverflow.com/questions/15313418/javascript-assert
https://www.wikiwand.com/en/Assertion_(software_development)

進行最後一個測試：

新增 test/controllers/store_controller_test.rb
```rb
test "markup needed for store.coffee is in place" do 
  get store_index_url
  assert_select '.store .entry > img', 3 
  assert_select '.entry input[type=submit]', 3
end
```

如果有人試圖改變頁面中 HTML 的運作邏輯，我們就會得到警告而能在上 production 前修改好。要注意的是，`:submit` 是 jQuery 專屬的 CSS 外掛，直接在測試中輸入 `input[type=submit]` 就搞定了。

讓測試能夠與時俱進是維護程式重要的一環，而 Rails 把這件事變簡單了。敏捷開發的程序猿把測試安排在開發過程中，有些甚至還會在寫第一行代碼前就先把測試寫好。

到此，我們就完成了整個 Task F: Add a Dash of Ajax. 

---

最後，有幾點小提示：
1. 如果要從事許多 Ajax 的開發，最好是多熟悉瀏覽器的 debugging 方式以及 DOM 的 inspectors, 比如說 Firefox 的 Firebug, IE 的 Developer Tools, Google Chrome 的 Developer Tools, Safari 的 Web Inspector, 或 Opera 的 Dragonfly.
2. Firefox 的 NoScript 外掛讓 Firefox 能一鍵切換 JavaScript/no JavaScript 的環境。