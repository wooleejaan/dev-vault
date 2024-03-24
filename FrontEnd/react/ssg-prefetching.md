# SSG와 Prefetching으로 블로그 성능 높이기

next js app router(v13~)를 사용해 [개인 블로그](https://github.com/wooleejaan/august-archive)를 제작했다.

### SSR로 블로그를 만들면 안 되는 이유

Next.js로 페이지를 만들고, Notion api를 사용해서 Notion에서 작성한 글을 로드해오는 방식으로 블로그를 제작했었다. 원래 블로그는 SSG로 만드는 게 당연하지만, SSR과 SSG의 속도 차이가 어느 정도인지 궁금했다.

SSR 방식으로 페이지를 만들고 배포를 하고 나니, 페이지 간 이동 시간이 최소 5초에서 심지어 느릴 때는 10초 이상 걸릴 때도 있었다. 로딩 애니메이션을 추가하는 정도로는 감출 수 없을 정도로 매우 느렸다.

### SSR에서 SSG로 변경하기

Next.js app router에서 렌더링 방식을 선택하는 건 엄청 쉽다.

먼저, notion database에서 페이지 데이터를 불러오는 로직을 아래와 같이 구현한다.

```js
// notion.helper.ts

export const getPages = (
  containProperty: string,
  pageSize = 3,
  startCursor?: string,
) =>
  notionClient.databases.query({
    filter: {
      and: [
        {
          property: 'Status',
          multi_select: {
            contains: 'published',
          },
        },
        {
          property: 'Status',
          multi_select: {
            contains: containProperty,
          },
        },
      ],
    },
    database_id: process.env.NOTION_DATABASE_ID,
    page_size: pageSize,
    start_cursor: startCursor,
  })
export const getPageContent = (pageId: string) =>
  notionClient.blocks.children
    .list({ block_id: pageId })
    .then((res) => res.results as BlockObjectResponse[])
```

그리고 이 notion 함수를 next api 안에서 호출한다.

```js
import { NextRequest, NextResponse } from "next/server";
import { getPageContent } from "@/libs/_shared/helpers/notion.helper";
export async function GET(req: NextRequest) {
  const detailId = req.nextUrl.searchParams.get("detailId") || "";
  const detail = await getPageContent(detailId);
  return NextResponse.json(detail);
}
```

그리고 일반적으로 api 호출하듯이 핸들러를 제작해준다.

- cache: force-cache가 SSG
- cache: no-store이 SSR
- revalidate을 설정하는 게 ISR

에 해당하는 옵션이라고 생각하면 쉽다.

단순하게 한 줄 옵션 코드만 추가해주면 렌더링 방식을 선택할 수 있어 매우 편리하다.

````js
/**
*
* @description SSG 적용된 API Handler
* @example
* ```bash
* # how to revalidate
* GET `/api/revalidate?path=/&secret=${process.env.MY_SECRET_TOKEN}`
* ```
*/
const getPageContentHelper = cache(async <T>(pageId: string): Promise<T> => {
	const res = await fetch(
		`${process.env.NEXT_PUBLIC_BASE_URL}/api/notion/content?detailId=${pageId}`,
		{
			// cache: 'no-store',
			cache: 'force-cache',
			// next: { revalidate: 86400 },
		},
	)

	if (!res.ok) {
		// This will activate the closest `error.ts` Error Boundary
		throw new Error('Failed to fetch data')
	}
	const data = res.json()
		return data
	})
	const getPageDetail = async (slug: string, property: string) => {
	const page = await getPageBySlugHelper<PageDetailHelperResponse>(
		slug,
		property,
	)

	if (!page) notFound()
	const polishedProps =
		page.properties as unknown as PartialDetailPageObjectResponseMore
	const info = {
		createdTime: dateConverter(page.created_time),
		subTitle: polishedProps?.SubTitle.rich_text[0].plain_text,
		category: polishedProps?.Category.multi_select.map((tag) => tag.name),
		slug: polishedProps?.Slug.rich_text[0].plain_text,
		title: polishedProps?.Title.title[0].plain_text,
	}

	const content = await getPageContentHelper<
		Array<BlockObjectResponse & BlockObjectMoreResponse>>(page.id)
	const updatedContent = await Promise.all(
		content.map(async (c, index) => {
		if (c.type === 'image') {
			const customUrl = await convertToGithubImage(
				info.slug,
				index,
				c.image.file.url,
			)

			// eslint-disable-next-line no-param-reassign
			c.image.file.url = customUrl
		}

		return c
		}),
	)

	const notionRenderer = new NotionRenderer({
		client: notionClient,
	})

	notionRenderer.use(hljsPlugin({}))
	notionRenderer.use(bookmarkPlugin(undefined))
	const html = await notionRenderer.render(...updatedContent)
	return { html, info }
}
````

### Link 태그의 prefetching

Next.js의 Link 태그는 prefetch 기능을 기본적으로 제공해준다.

물론, prefetch={false}로 해제할 수도 있다. 해제하면 hover 시에 prefetching해준다고 하는데, 블로그 데이터의 경우 hover 시에 가져오는 것도 꽤 느려서 기본값을 사용했다.

```jsx
import Link from "next/link";

{
  /* ... */
}
<ol className={cx("previewContainer")}>
  {section.map(({ slug, title }) => (
    <li key={title}>
      <Link href={`/${sectionType}/${slug}`} className={cx("link")}>
        <p className={cx("list")}>{title}</p>
      </Link>
    </li>
  ))}
</ol>;
```

아래 이미지를 보면 Link로 구현된 태그들에 대해 prefetching해오는 것을 확인할 수 있다.

### SSG인데 revalidate도 사용하려면?

노션으로 글을 작성하면, 배포된 블로그에 바로 반영되지 않는다.<br>
`cache: force-cache` 옵션을 줘서 SSG로 구현했기 때문이다.

이때 주기적이지는 않으면서, 업로드 시에만 revalidate을 해주고 싶을 때가 있다.<br>
이럴 때 next.js에서 revalidate 기능을 쉽게 구현할 수 있다.

```js
// app/api/revalidate/route.ts

import { revalidatePath } from "next/cache";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const secret = request.nextUrl.searchParams.get("secret");
  if (secret !== process.env.MY_SECRET_TOKEN) {
    return new NextResponse(
      JSON.stringify({
        message: "Invalid Token",
      }),
      {
        status: 401,
        statusText: "Unauthorized",
        headers: {
          "Content-Type": "application/json",
        },
      }
    );
  }

  const path = request.nextUrl.searchParams.get("path") || "/";
  revalidatePath(path);
  return NextResponse.json({
    revalidated: true,
  });
}
```

위와 같이 revalidate api를 구현해두고, 토큰이 확인되면 next/cache의 revalidatePath 함수를 실행하도록 하면, 웹 전체를 revalidate을 쉽게 할 수 있어 편하다.
