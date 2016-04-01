---
title: Filling Laravel Relationships
date: 2016-04-01 16:55:01
description: .fil(:allthethings:)
---

Laravel has a pretty great ORM behind it named Eloquent. You can model just about any data and relationship you need to within it's expressive syntax. Take a look at the following,

```php
class Requirement {

  public function milestone()
  {
    return $this->belongsTo(Milestone::class);
  }

}
```

The above will associate a requirement with a milestone through the `milestone_id` field on the `requirements` table. Using this relationship is where it becomes paticularlly helpful. Within a `.twig` template you may have,

```twig
<td>{{ requirement.title }}</td>
<td>{{ requirement.milestone.title|default('None') }}</td>
```

So far this is what you'd expect out of an ORM. Where I struggle with Eloquent time and time again is trying to set these relationships during a `save()`. For example, in Rails you can do something like this (usually within a `select` call but written out here for clarity),

```erb
<select name="requirement[milestone]">
  <option value=""></option>
  <% @milestones.each do |milestone| %>
  <option value="<%= milestone.id %>"><%= milestone.title %></option>
  <% end %>
</select>
```

Then, within your controller you'd save all this with,

```ruby
@requirement.update(params[:requirement])
```

Easy and expected. Eloquent has no such functionality. You have to manually call,

```php
$requirement->milestone()->associate($request->input('requirement.milestone'));
```

This doesn't scale. I'd rather not write this out for each relationship on my requirement model. The best approach I've seen to solve this is to use the observer pattern and hook into the `Requirement::saved()` event. I'm doing this with a trait and an observer class. The trait is responsible for setting everythign up and the observer is the one that actually manages the relationship. The trait looks like this,

```php
trait MilestoneUpdateTrait {

  public $milestoneUpdateId;

  /**
   * Boot the model and run and user defined actions here.
   */
  public static function bootMilestoneUpdateTrait()
  {
    parent::observe(new MilestoneUpdateObserver);
  }

  /**
   * Store the modle we're trying to relate to
   */
  public function setMilestoneAttribute($value)
  {
    $this->milestoneUpdateId = $value;
  }

}
```

It doesn't do much other than register the observer and intercept the `fill(['milestone' => ...])` call. The reason we need to intercept the `setMilestoneAttribute` call is because we don't want `milestone` to go into the model's attributes and attempt to save it to a `requirements.milestone` field.

The observer, then, looks like this,

```php
class MilestoneUpdateObserver {

  public function saved($model)
  {
    if ($model->milestoneUpdateId) {
      $model->milestone()->associate($model->milestoneUpdateId);
      $model->milestoneUpdateId = null;
      $model->save();
    }
  }

}
```

Which basically just checks if we're trying to set a milestone and then calls the correct Eloquent method to save the association.

I haven't dealt with un-setting the association yet. That'll have to be worked out next. But for now this allows me to use Rails-style HTML with a simple `->fill()` call in my controller.
