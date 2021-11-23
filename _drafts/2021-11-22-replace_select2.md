---
layout: post
title: Replacing Select2 with Tom Select + Stimulus
---
We all used [Select2](https://select2.org/). We all depended on it for a long time, for all our Select/Autocomplete needs. But it's been showing signs of aging for quite a while, and it's one of the last libraries that still keeps me tied to [jQuery](https://love2dev.com/blog/jquery-obsolete/).

It was time to let go.

After quickly [evaluating](https://github.com/brianvoe/slim-select) a [few](https://github.com/wolffe/tail.select.js) [alternatives](https://github.com/selectize/selectize.js), I decided to take a closer look at [Tom Select](https://tom-select.js.org/).

> Tom Select was forked from selectize.js with the goal of modernizing the code base, decoupling from jQuery, and expanding functionality.

Well, that works for me.

And since it was renovating time, I also decided to consolidate all that JS used to make Select2 work with Ajax, filtering, etc. in a few [Stimulus](https://stimulus.hotwired.dev/) controllers.

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

I just imported `TomSelect` and created the instance on connect

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

Worked like a charm! For simple cases, when all you need is a select/options backed solution, that is all it takes.

---
## Autocomplete with Ajax

Sometimes, when your list is too large, you may prefer a remote approach.

Keeping the simplicity in mind, I wanted somenthing like:

{% highlight erb %}

<%= f.select :search_city, [], {},
              placeholder: 'Type to search',
              data: {
                controller: 'ts--search',
                ts__search_url_value: autocomplete_cities_path
              } %>

{% endhighlight %}

The remote endpoint just returns a JSON Array with the matching results

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

Thanks to Tom Select support for [remote](https://tom-select.js.org/examples/remote/) data, and using the great [@rails/request.js](https://github.com/rails/request.js), the actual implementation ended up quite straightforward.


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

    if(response.ok) {
      callback(await response.json)
    } else {
      console.log(response)
      callback()
    }
  }

}

{% endhighlight %}