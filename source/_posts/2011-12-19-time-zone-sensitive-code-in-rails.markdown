---
layout: post
title: "Time Zone Sensitive Code in Rails"
date: 2011-12-19 09:56
comments: true
categories: [ruby, rails]
---
At [Goldstar](http://www.goldstar.com), we help people go out more to live entertainment across several time zones. When we list a new show in New York City and it's supposed to be displayed on our site at midnight on December 1st, we want it to be displayed when it's midnight in New York City on December 1st, and not wait until it's midnight in Los Angeles. Rails hasn't done a great job of making it obvious how to do this, so I hope to make it obvious how we've done it successfully using Rails.

**TL;DR:** if you're doing time zone sensitive stuff, use ```Time.zone```, followed by ```.now``` or ```.now.to_date```; or use the Rails helpers like ```1.hour.ago```.

Since we are based in California, we use Pacific Time in our app. This is set in our app's config/environment.rb (config/application.rb for Rails 3.x):
```
config.time_zone = 'Pacific Time (US & Canada)'
```

Our time zone is now set to Pacific Time, which is evidenced by:

```
  > Time.zone
 => #<ActiveSupport::TimeZone:0x1c865d0 @tzinfo=#<TZInfo::DataTimezone: America/Los_Angeles>, @utc_offset=-28800, @current_period=nil, @name="Pacific Time (US & Canada)">
```

However, Time.now and Date.today use the OS's time zone. Since I am in Mountain Time, I get -0700. Pacific Time folks wouldn't think anything was amiss:
```
  > Time.now
 => Wed Dec 14 09:52:36 -0700 2011
  > Date.today.to_time
 => Wed Dec 14 00:00:00 -0700 2011
```

### How to Use the App's Time Zone
If you want a date or time in the app's current time zone, use Time.zone.now or the Rails helpers like 1.hour.ago:
```
  > Time.zone.now
 => Wed, 14 Dec 2011 08:52:36 PST -08:00
  > 1.hour.ago
 => Wed, 14 Dec 2011 07:52:36 PST -08:00
```

You can switch Rails' time zone if you know the right incantation (for a full list, [look here](http://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html)):
```
  > Time.zone = 'America/New_York'
 => "America/New_York"
```

You'll notice that Time.zone.now uses Rails' time zone correctly when you've changed it:
```
  > Time.zone.now
 => Wed, 14 Dec 2011 11:52:37 EST -05:00
```

In case you're curious, you can change the OS's time zone if you know the right incantation as well:
```
  > ENV['TZ'] = 'EST'
 => "EST"
  > Time.now
 => Wed Dec 14 11:56:23 -0500 2011
```

One problem we kept having at Goldstar was a failing build after certain hours of the day, since our app and CI server run in Pacific Time but some of our tests test shows in other time zones.

### Homework
So with your new-found time zone knowledge, here's an exercise for you. Can you spot the bug with this spec:

``` ruby
describe "thinger" do
  before :each do
    @event = Factory(:event)
  end

  it "should return the thinger for this event if its thinger_on is today" do
    Time.use_zone(@event.timezone) do
      @thinger = Thinger.create!(:event => @event, :thinger_on => Time.now.to_date)
    end
    @event.thinger.should == @thinger
  end
end
```

ANSWER KEY: :thinger_on is using Time.now.to_date, which after a certain hour of the day, depending on your OS's time zone, will fail. Totally lame. Time.zone.now.to_date is what was intended here.
