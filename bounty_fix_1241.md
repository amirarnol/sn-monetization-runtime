إليك تحليل البيانات وحل برمجي احترافي ومكتوب بلغة **Python** (مطابق لمعايير PEP 8 وخالٍ من الأخطاء Lint-free) لقراءة هذه السجلات وتحليل بيانات مكافآت **Stacker News (SN)** واستخراج رابط المنشور وتفاصيله بدقة.

---

### 1. تحليل حقول السجل (Data Parsing)

البيانات المرسلة تمثل سطراً مجزءاً بعلامة الجدولة (`Tab-separated values - TSV`):

| الحقل | القيمة | الوصف |
| :--- | :--- | :--- |
| **Item ID** | `1586915` | المعرّف الرقمي للمنشور على Stacker News |
| **Territory / Sub** | `Stacker_Sports` | القسم التابع له المنشور |
| **Upvotes** | `3` | عدد التصويتات الإيجابية |
| **Bounty (Sats)** | `1130` | قيمة المكافأة الحالية بالساتوشي |
| **Max Bounty / Cap** | `2100` | الحد الأقصى للمكافأة |
| **Comments** | `10` | عدد الردود الحالية |
| **Weight / Score** | `23.0` | تقييم الخوارزمية للمنشور |
| **Author ID** | `232181` | معرّف الكاتب |
| **Author Rep** | `4270` | سمعة الكاتب أو نشاطه |
| **Feeds** | `recent@...\|top@...` | موجزات الظهور |
| **Flags** | `OPEN_BOUNTY, HOT, ...` | حالة المنشور والفرص التنافسية |
| **Title** | `Weekly Random Sports Pick 'em` | عنوان المنشور |

---

### 2. الكود البرمجي (Python 3.10+)

```python
"""Stacker News Bounty Alert Parser and Processor."""

from __future__ import annotations

from dataclasses import asdict, dataclass
import json
import logging
from typing import Optional

logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")
logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class BountyAlert:
    """Represents a structured Stacker News Open Bounty event."""

    item_id: int
    territory: str
    upvotes: int
    bounty_sats: int
    max_bounty_sats: int
    comments_count: int
    score: float
    author_id: int
    author_reputation: int
    feeds: list[str]
    flags: list[str]
    title: str

    @property
    def url(self) -> str:
        """Construct the direct Stacker News post URL."""
        return f"https://stacker.news/items/{self.item_id}"

    @property
    def is_active_bounty(self) -> bool:
        """Check if bounty is still open based on flags."""
        return "OPEN_BOUNTY" in self.flags

    @classmethod
    def from_tsv(cls, raw_line: str) -> Optional[BountyAlert]:
        """Parse a raw tab-separated line into a BountyAlert instance."""
        fields = [f.strip() for f in raw_line.strip().split("\t")]
        if len(fields) < 12:
            logger.error(
                "Invalid record: expected at least 12 fields, received %d",
                len(fields),
            )
            return None

        try:
            return cls(
                item_id=int(fields[0]),
                territory=fields[1],
                upvotes=int(fields[2]),
                bounty_sats=int(fields[3]),
                max_bounty_sats=int(fields[4]),
                comments_count=int(fields[5]),
                score=float(fields[6]),
                author_id=int(fields[7]),
                author_reputation=int(fields[8]),
                feeds=fields[9].split("|") if fields[9] else [],
                flags=fields[10].split(",") if fields[10] else [],
                title=fields[11],
            )
        except (ValueError, TypeError) as err:
            logger.error("Failed to parse numeric/type fields: %s", err)
            return None

    def to_json(self) -> str:
        """Serialize instance to formatted JSON string."""
        payload = asdict(self)
        payload["url"] = self.url
        return json.dumps(payload, indent=2, ensure_ascii=False)


def handle_bounty_event(raw_data: str) -> None:
    """Process incoming bounty string and output parsed result."""
    alert = BountyAlert.from_tsv(raw_data)
    if alert is None:
        raise ValueError("Could not parse bounty alert payload.")

    logger.info("Successfully parsed item #%d: '%s'", alert.item_id, alert.title)
    print(alert.to_json())


if __name__ == "__main__":
    RAW_INPUT = (
        "1586915\tStacker_Sports\t3\t1130\t2100\t10\t23.0\t"
        "232181\t4270\trecent@Stacker_Sports|top@Stacker_Sports\t"
        "OPEN_BOUNTY,HOT,SELF_POST_OPP\tWeekly Random Sports Pick 'em"
    )

    handle_bounty_event(RAW_INPUT)
```

---

### 3. مخرجات المعالجة (JSON Output)

عند تشغيل الكود أعلاه، تكون النتيجة كائن JSON معالجاً وجاهزاً للاستخدام في الأنظمة المؤتمتة:

```json
{
  "item_id": 1586915,
  "territory": "Stacker_Sports",
  "upvotes": 3,
  "bounty_sats": 1130,
  "max_bounty_sats": 2100,
  "comments_count": 10,
  "score": 23.0,
  "author_id": 232181,
  "author_reputation": 4270,
  "feeds": [
    "recent@Stacker_Sports",
    "top@Stacker_Sports"
  ],
  "flags": [
    "OPEN_BOUNTY",
    "HOT",
    "SELF_POST_OPP"
  ],
  "title": "Weekly Random Sports Pick 'em",
  "url": "https://stacker.news/items/1586915"
}
```