---
layout: post
title: Replacing Select2 with Tom Select + Stimulus
date: 2021-11-25 15:12:40 -0300
tags: Stimulus JS Rails
---
We all used [Select2](https://select2.org/). We all depended on it for a long time, for all our Select/Autocomplete needs. But it's been showing signs of aging for quite a while, and it's one of the last libraries that still keeps me tied to [jQuery](https://love2dev.com/blog/jquery-obsolete/).

It was time to let go.

After quickly [evaluating](https://github.com/brianvoe/slim-select) a [few](https://github.com/wolffe/tail.select.js) [alternatives](https://github.com/selectize/selectize.js), I decided to take a closer look at [Tom Select](https://tom-select.js.org/).

> Tom Select was forked from selectize.js with the goal of modernizing the code base, decoupling from jQuery, and expanding functionality.

Well, that looks good to me.

And since it was renovating time, I also decided to consolidate all that JS used to make Select2 work with Ajax, filtering, etc. in a few [Stimulus](https://stimulus.hotwired.dev/) controllers.

<div class="message">
All the examples shown here are coded with a <b>Rails</b> app in mind, but they can easily be adapted for any other stack.
</div>

## The simple case

I wanted to make it as simple as possible to use the solution from the HTML code. The simplest solution would be just adding a `data-controller` to a select tag.

{% highlight erb %}

<%= f.select :city,
              City.to_select,
              { include_blank: true },
              data: {
                controller: 'ts--select'
              } %>

{% endhighlight %}

So, I installed **tom-select** and created a `ts--select` controller, using the new generator from [stimulus-rails](https://github.com/hotwired/stimulus-rails):
{% highlight bash %}
$ yarn add tom-select

$ rails g stimulus ts/select
{% endhighlight %}

I just imported `TomSelect` and created the instance on connecting

{% highlight js %}
import { Controller } from "@hotwired/stimulus"
import TomSelect      from "tom-select"

// Connects to data-controller="ts--select"
export default class extends Controller {
  connect() {
    new TomSelect(this.element)
  }
}
{% endhighlight %}

It worked like a charm! For simple cases, when all you need is a select/options backed solution, that is all it takes.

---
## Autocomplete with Ajax

Sometimes, when your list is too large, you may prefer a remote approach.

Keeping simplicity in mind, I wanted something like:

{% highlight erb %}

<%= f.select :search_city, [], {},
              placeholder: 'Type to search',
              data: {
                controller: 'ts--search',
                ts__search_url_value: autocomplete_cities_path
              } %>

{% endhighlight %}

The remote endpoint only needs to return a JSON Array with the matching results.

{% highlight ruby %}

# app/controllers/cities_controller.rb

def autocomplete
  list = City.order(:name)
             .where("name ilike :q", q: "%#{params[:q]}%")

  render json: list.map do |u|
    {
      text: u.name,
      value: u.id
    }
  end

end

{% endhighlight %}

Thanks to Tom Select's support for [remote](https://tom-select.js.org/examples/remote/) data and using the handy [@rails/request.js](https://github.com/rails/request.js), the actual implementation ended up quite straightforward.


{% highlight js %}
// app/javascript/controllers/ts/search_controller.js

import { Controller } from "@hotwired/stimulus"
import { get }        from '@rails/request.js'
import TomSelect      from "tom-select"

// Connects to data-controller="ts--search"
export default class extends Controller {
  static values = { url: String }

  connect() {

    var config = {
      plugins: ['clear_button'],
      valueField: 'value',
      load: (q, callback) => this.search(q, callback)
    }

    new TomSelect(this.element, config)
  }

  async search(q, callback) {
    const response = await get(this.urlValue, {
      query: { q: q },
      responseKind: 'json'
    })

    if (response.ok) {
      const list = await response.json
      callback(list)
    } else {
      console.log(response)
      callback()
    }
  }

}

{% endhighlight %}

---

## Filtering other Select

Another very common use case is the need to filter the options of a select based on the selected value of another one. In this case, our controller has to include both selects. The markup I imagined for such a task would be something like:

{% highlight erb %}
<div data-controller="ts--filter"
     data-ts--filter-url-value="<%= filter_cities_path %>">

  <%= f.select :state,
                City.states_to_select,
                { include_blank: true },
                data: {
                  ts__filter_target: 'filter'
                } %>

  <%= f.select :filter_city, [], {},
                data: {
                  ts__filter_target: 'other'
                } %>

</div>
{% endhighlight %}

Again, the remote endpoint returns the filtered results

{% highlight ruby %}

# app/controllers/cities_controller.rb

def filter
  list = City.order(:name)
             .where(state: params[:filter])

  render json: list.map { |u| { text: u.name, value: u.id } }
end

{% endhighlight %}

The stimulus controller is somewhat similar to the previous one, but with enough differences to require a new one

{% highlight js %}

// app/javascript/controllers/ts/filter_controller.js

import { Controller } from "@hotwired/stimulus"
import { get }        from "@rails/request.js"
import TomSelect      from "tom-select"

export default class extends Controller {
  static targets = [ "filter", "other" ]
  static values  = { url: String }

  connect() {

    this.filterTarget.addEventListener('change', ev => {
      if (this.selectedFilter)
        this.fetchItems()
      else
        this.clearItems()
    })

    if (this.selectedFilter) this.fetchItems()
  }

  async fetchItems() {
    const response = await get(this.urlValue, {
      query: { filter: this.selectedFilter },
      responseKind: 'json'
    })

    if (response.ok)
      this.setItems(await response.json)
    else
      console.log(response)
  }

  setItems(items) {
    this.clearItems()
    this.tomSelect.addOptions(items)
  }

  clearItems() {
    this.tomSelect.clear()
    this.tomSelect.clearOptions()
  }

  get selectedFilter() {
    return this.filterTarget.value
  }

  get tomSelect() {
    this._tomSelect ||= new TomSelect(this.otherTarget, {
      plugins: [ 'clear_button']
    })

    return this._tomSelect
  }

}

{% endhighlight %}

It checks for a selected value every time the State changes and on connecting. Then it fetches or clears the results accordingly.

---
## Final Thoughts

Tom Select is a very useful library to implement advanced behaviour in select tags, and can replace select2 with advantages. We've seen examples of 3 different scenarios. Using Stimulus allows us to implement advanced funcionalities in the select, while keeping the html simple and enabling reuse of the code across different pages.

<div class="message">
  You can find a functional demo of this concept here <br><b><a href="https://tom-select.herokuapp.com/">https://tom-select.herokuapp.com/</a></b>
  <br><br>
  The source code can be found at <br><b><a href="https://github.com/CoolRequest/tom-select-demo/">https://github.com/CoolRequest/tom-select-demo/</a></b>
</div>