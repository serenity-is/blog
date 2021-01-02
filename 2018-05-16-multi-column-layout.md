---
title: Multi Column Layouts without CSS
description: A philosopher said "change is the only constant in life". By this logic, Serenity should not be a constant, thus it changes a lot. If in doubt, just take a peek at our change log (https://github.com/serenity-is/Serenity/blob/master/CHANGELOG.md). Even though we write about them in that file, so many cool new features goes unnoticed, or might not be fully understood from the short description in the log. One of the features that we'd like to talk about is the one we added to simplify multi column layouts, which is modelled after Bootstrap grid system / column classes.
author: Volkan Ceylan
---

## CSS Based Multi Column Layouts

Serenity and Bootstrap meet later in life, so originally Serenity form layouts weren't based on Boostrap or any other popular grid system.

Initially, to make nice looking forms with multi column layouts, we started with simple CSS floats which wasn't so intuitive, hard to scale / make responsive and was error prone. Later, thanks to the standardization of flexbox we redesigned our forms and you now don't have to mess much with floats, and hard coded editor sizes.

Before the attribute based layout system we'll talk about, we used to write CSS like below to create a multi column form:

```css
.s-BasicSamples-MultiColumnResponsiveDialog {
    > .size { width: 850px; height: 650px; }
    .caption { width: 130px; }

    @media (min-width: 768px) {
        .field.OrderDate, .field.RequiredDate, .field.ShipName, .field.ShipAddress, 
        .field.ShipCity, .field.ShipRegion, .field.ShipPostalCode, .field.ShipCountry {
            flex: auto;
            min-width: 50%;
        }
        
        .field.ShippedDate, .field.ShipVia, .field.Freight { 
            flex: auto;
            min-width: 33%;
            .caption { width: 90px; } 
        }
    }
}
```

This CSS code above is from a sample which we had in Serene demo (it's removed now). Here we demonstrate how you could make Order form in Northwind multi column, e.g. more than one field in one line.

OrderDate, RequiredDate, ShipName... columns are placed as two fields per line, while ShippedDate, ShipVia, and Freight are three fields per line.

We use media queries to make this rule take effect only in large enough devices with screen estate that can handle multiple fields per line.

The primary problem with this design is maintainability. We had to hardcode field names here which is prone to typos. When you add / remove / rename fields in the order form, you would also apply changes here. Another issue is if you wanted to reuse this form layout in another dialog, you would have to add it to css rules.

What if we wanted to support different layouts for more device types, e.g. a basic phone, phablet, tablet, notebook, desktop etc? We would have to add more media rules and write more CSS.

## Layout Attributes (added in Serenity 3.3.12)

Using new layout related attributes in Serenity, we can now directly apply layout above in OrderForm.cs:

```cs
public class OrderForm
{
    [HalfWidth]
    public DateTime OrderDate { get; set; }
    [HalfWidth]
    public DateTime RequiredDate { get; set; }

    [OneThirdWidth]
    public DateTime ShippedDate { get; set; }
    [OneThirdWidth]
    public Int32 ShipVia { get; set; }
    [OneThirdWidth]
    public Decimal Freight { get; set; }

    [HalfWidth]
    public String ShipName { get; set; }
    [HalfWidth]
    public String ShipAddress { get; set; }
    [HalfWidth]
    public String ShipCity { get; set; }
    [HalfWidth]
    public String ShipRegion { get; set; }
    [HalfWidth]
    public String ShipPostalCode { get; set; }
    [HalfWidth]
    public String ShipCountry { get; set; }
}
```

[HalfWidth] and [OneThirdWidth] attributes both derive from [FormWidth] attribute. Actually:

```cs
[HalfWidth] = [FormWidth("col-sm-6")]
[OneThirdWidth] = [FormWidth("col-sm-4")]
```

Spotted Bootstrap column classes there? They are the css classes that gets added to relevant form fields when rendered.

Ok this looks fine, but we can do better, why repeat [HalfWidth] for every property while we can do it once?

```cs
public class OrderForm
{
    [HalfWidth(UntilNext = true)]
    public DateTime OrderDate { get; set; }
    public DateTime RequiredDate { get; set; }

    [OneThirdWidth(UntilNext = true)]
    public DateTime ShippedDate { get; set; }
    public Int32 ShipVia { get; set; }
    public Decimal Freight { get; set; }

    [HalfWidth(UntilNext = true)]
    public String ShipName { get; set; }
    public String ShipAddress { get; set; }
    public String ShipCity { get; set; }
    public String ShipRegion { get; set; }
    public String ShipPostalCode { get; set; }
    public String ShipCountry { get; set; }
}
```

This is functionally equivalent but cleaner. When you pass *UntilNext* = true to any form width attribute, it applies its sizing to following fields until it sees another form width attribute. This works similar to *[Category]* attribute we use in forms. 

We also have following form width attributes available, some of which are added by community members:

```
[QuarterWidthAttribute] = [FormWidth("col-md-3 col-sm-6")]
[ThreeQuarterWidth] = [FormWidth("col-md-9")]
[TwoThirdWidth] = [FormWidth("col-sm-8")]
[MediumThirdLargeQuarterWidth] = [FormWidth("col-md-4 col-lg-3")]
[MediumQuarterWidth] = [FormWidth("col-md-3")]
[MediumHalfWidth] = [FormWidth("col-md-6")]
[MediumHalfLargeWidth] = [FormWidth("col-md-6 col-lg-4")]
[MediumHalfLargeQuarterWidth] = [FormWidth("col-md-6 col-lg-3")]
[ResetFormWidth] = [FormWidth(null)] // used to reset form width
```

You're not limited to these, and can use any custom grid layout class combination you like using [FormWidth("...")] attribute, even can define your own subclasses.

## Controlling Label Width w/o Css

If you look at prior CSS sample, you might notice that we set label width for ShippedDate, ShipVia and Freight fields to *90px*. Can we do same in Form.cs? Simple:

```cs
public class OrderForm
{
    [HalfWidth(UntilNext = true), LabelWidth(130, UntilNext = true)]
    public DateTime OrderDate { get; set; }
    public DateTime RequiredDate { get; set; }

    [OneThirdWidth(UntilNext = true), LabelWidth(90, UntilNext = true)]
    public DateTime ShippedDate { get; set; }
    public Int32 ShipVia { get; set; }
    public Decimal Freight { get; set; }

    [HalfWidth(UntilNext = true), LabelWidth(130, UntilNext = true)]
    public String ShipName { get; set; }
    public String ShipAddress { get; set; }
    public String ShipCity { get; set; }
    public String ShipRegion { get; set; }
    public String ShipPostalCode { get; set; }
    public String ShipCountry { get; set; }
}
```

We also have [ResetLabelWidth] in case you want to reset label widths for following fields.

## Line Breaking

What if you wanted an editor half size, but put only one in line, and wrap next?

```cs
public class OrderForm
{
    [HalfWidth(UntilNext = true), LabelWidth(130, UntilNext = true)]
    public DateTime OrderDate { get; set; }
    [FormCssClass("line-break-sm")]
    public DateTime RequiredDate { get; set; }
}
```

Here both OrderDate and RequiredDate will be half size and will be on their own line (not same line) in small (>768px) or larger devices.

## Conclusion

Even though we think CSS should be used for styling and (most) layout, in some cases like designing a form, it is more intiutive to use layout attributes directly in code. 

We hope that it'll help you to design complex forms with relative ease.

We'll keep on writing about more features Lost In Change Log.