---
layout: post
title: "Генерация PDF с помощью Prawn"
date: 2013-06-21 18:17
comments: true
categories:
    - Rails
    - Ruby
---

* TOC
{:toc}

Введение
========

Динамическое создание PDF &mdash; довольно распространенное задача.
Для Ruby существует несколько библиотек для генерации с PDF-файлов,
вот некоторые из них:

- [PDFkit][pdfkit] &mdash; библиотека для конвертации HTML в PDF. Использует
  shell-утилиту [wkhtmltopdf][wkhtmltopdf], которая, в свою очередь,
  использует webkit в качестве движка рендеринга;

- [Wicked pdf][wickedpdf] &mdash; тоже конвертация HTML &rarr; PDF,
  тоже [wkhtmltopdf][wkhtmltopdf];

- [Kitabu][kitabu] &mdash; генерация PDF и e-Pub из Textile, Markdown
  или HTML;

- [Prawn][prawn] &mdash; библиотека для создания PDF-файлов,
  написанная на чистом Ruby.

Таким образом, видны две тенденции: генерация PDF из какого-нибудь
языка разметки либо использование DSL для генерации.

Мой выбор пал на Prawn, о нем и будет рассказано в остальной части
поста.

<!-- more -->

Установка и настройка
=====================

Установка проста: добавьте следующую строку в `Gemfile`:

``` ruby
gem 'prawn'
```

Затем нужно добавить в `config/application.rb` автозагрузку файлов из
директории `/app/pdfs/`, в которой мы будем хранить классы, создающие
PDF-файлы:

``` ruby
module Biotriz
  class Application < Rails::Application
    config.autoload_paths << "#{Rails.root}/app/pdfs"
    # ... skipped ...
  end
end
```

Далее зарегистрируем Mime-тип `:pdf` в
`/config/initializers/mime_types.rb`:

``` ruby
Mime::Type.register 'application/pdf', :pdf
```

Это позволит нам указывать тип `:pdf` в методе `send_data`, а также
использовать его в блоке `respond_to`.

Использование
=============

Код класса, создающего PDF:

``` ruby app/pdfs/search_results.rb
# encoding: utf-8

require "prawn/measurement_extensions"

class SearchResults < Prawn::Document
  include Rails.application.routes.url_helpers
  include ActionDispatch::Routing::PolymorphicRoutes

  def initialize(query, entries, host)
    super()
    pad(20) do
      font("Helvetica", size: 20, style: :bold) do
        text "Search Results (#{Time.zone.now.strftime('%d/%m/%Y %H:%M')})"
      end
    end

    pad_bottom(20) do
      font("Helvetica", size: 14, style: :bold) do
        pad_bottom(5) do
          text "Search Query"
        end
      end

      query_output = []
      first = false

      query.try(:each) do |item|
        color = item[:color][1..-1]
        cls = item[:model].classify.constantize

        item[:values].each do |value|
          query_output << {
            text: "[#{value.title}]",
            link: polymorphic_url(value, host: host)
          }

          if not first
            query_output << { text: "   " }
            first = false
          end
        end
      end

      formatted_text query_output, leading: 10
    end

    pad_bottom(20) do
      font("Helvetica", size: 14, style: :bold) do
        pad_bottom(5) do
          text "Search Results"
        end
      end
      entries.each do |entry|
        pad_bottom(10) do
          font("Helvetica", style: :bold) do
            formatted_text [{text: entry.title, link: polymorphic_url(entry, host: host)}]
          end
        end

        pad_bottom(15) do
          text entry.description, indent_paragraphs: 1.5.cm
        end
      end
    end
  end
end
```

Особо комментировать этот код не буду, как по мне, он говорит сам за
себя; желающие могут обратиться к документации [на сайте][prawn], там
же есть мануал.

Код действия, формирующего и отсылающего PDF-файл клиенту:

``` ruby
  def save
    respond_to do |format|
      # ... skipped ...
      format.pdf do
        query = restore_query params[:query]
        reverted_query = revert_query query: query
        host = request.host_with_port
        entries = # ... skipped ...
        send_data SearchResults.new(reverted_query, entries, host).render,
                  filename: "search.pdf",
                  type: :pdf,
                  disposition: "attachment"
      end
    end
  end
```

Заключение
==========

С помощью библиотеки Prawn можно относительно легко генерировать простые
(и не очень) PDF-файлы. Возможно, подход, выбранный создателями
библиотеки (DSL), не самый оптимальный, но со своей задачей она
справляется на отлично.

Однако, я даю себе слово попробовать и подход с генерацией PDF из
файлов, написанных на языках разметки ☺

[wkhtmltopdf]: http://code.google.com/p/wkhtmltopdf/

[pdfkit]: https://github.com/pdfkit/pdfkit

[wickedpdf]: https://github.com/mileszs/wicked_pdf

[kitabu]: https://github.com/fnando/kitabu

[prawn]: http://prawn.majesticseacreature.com/
