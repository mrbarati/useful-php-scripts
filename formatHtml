function formatHtml($html, $indentSize = 4, $useSpaces = true): string {
	$indent = $useSpaces ? str_repeat(' ', $indentSize) : "\t";

	$voidTags = ['area', 'base', 'br', 'col', 'embed', 'hr', 'img', 'input',
		'link', 'meta', 'param', 'source', 'track', 'wbr'];

	$inlineTags = ['a', 'span', 'b', 'i', 'em', 'strong', 'small', 'label',
		'code', 'mark', 'sub', 'sup', 'time', 'td', 'th', 'button'];

	$textInlineTags = ['h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'p', 'dt', 'dd', 'title', 'option'];

	$smartyBlockTags = ['foreach', 'for', 'while', 'section', 'literal', 'block',
		'capture', 'if', 'elseif', 'else', 'strip', 'nocache'];

	$preserveTags = ['pre', 'textarea', 'script', 'style'];
	$maxInlineLength = 80;

	if(empty($html) || !is_string($html)) {
		return $html ?? '';
	}

	$pattern = '~'
		. '(<!DOCTYPE[^>]*>)'
		. '|(<!--[\s\S]*?-->)'
		. '|({literal}[\s\S]*?{/literal})'
		. '|({\*[\s\S]*?\*})'
		. '|({[^}]+})'
		. '|(<[^>]+>)'
		. '|([^<{]+)'
		. '~iu';

	preg_match_all($pattern, $html, $matches, PREG_SET_ORDER);

	$tokens = [];
	foreach($matches as $match) {
		for($i = 1; $i < count($match); $i++) {
			if($match[$i] !== null && $match[$i] !== '') {
				$tokens[] = $match[$i];
				break;
			}
		}
	}

	$joined = implode('', $tokens);
	if(strlen($joined) < strlen($html)) {
		$remainder = substr($html, strlen($joined));
		if($remainder !== '') {
			$tokens[] = $remainder;
		}
	}

	$parsed = [];
	foreach($tokens as $token) {
		$t = trim($token);
		$type = 'text';
		$name = '';
		$smarty = false;
		$preserve = false;

		if(stripos($t, '<!DOCTYPE') === 0) {
			$type = 'doctype';
		}
		else if(str_starts_with($t, '<!--')) {
			$type = 'comment';
		}
		else if(stripos($t, '{literal}') === 0) {
			$type = 'smarty_preserve';
			$name = 'literal';
			$smarty = true;
			$preserve = true;
		}
		else if(str_starts_with($t, '{*')) {
			$type = 'smarty_comment';
			$smarty = true;
		}
		else if(preg_match('/^\{\/([a-zA-Z][a-zA-Z0-9]*)}/', $t, $m)) {
			$type = 'smarty_close';
			$name = strtolower($m[1]);
			$smarty = true;
		}
		else if(preg_match('/^\{([a-zA-Z][a-zA-Z0-9]*)/', $t, $m)) {
			$name = strtolower($m[1]);
			$smarty = true;
			if(in_array($name, $smartyBlockTags, true)) {
				$type = 'smarty_open';
			}
			else {
				$type = 'smarty_inline';
			}
		}
		else if(preg_match('/^\{\$/', $t) || preg_match('/^\{`/', $t) || preg_match('/^\{#/', $t)) {
			$type = 'smarty_inline';
			$smarty = true;
		}
		else if(preg_match('/^<\/([a-zA-Z][a-zA-Z0-9]*)\s*>/', $t, $m)) {
			$type = 'close';
			$name = strtolower($m[1]);
		}
		else if(preg_match('/^<([a-zA-Z][a-zA-Z0-9]*)/', $t, $m)) {
			$name = strtolower($m[1]);
			$isVoid = in_array($name, $voidTags, true) || str_ends_with($t, '/>');
			$type = $isVoid ? 'void' : 'open';
			if(in_array($name, $preserveTags, true)) {
				$preserve = true;
			}
		}

		$parsed[] = [
			'type' => $type,
			'name' => $name,
			'raw' => $token,
			'smarty' => $smarty,
			'preserve' => $preserve
		];
	}

	$ranges = [];
	for($i = 0; $i < count($parsed); $i++) {
		$type = $parsed[$i]['type'];
		if($type !== 'open' && $type !== 'smarty_open') {
			continue;
		}

		$name = $parsed[$i]['name'];
		$isSmarty = $parsed[$i]['smarty'];

		if(!$isSmarty && !in_array($name, $inlineTags, true) && !in_array($name, $textInlineTags, true)) {
			continue;
		}

		$depth = 1;
		$j = $i + 1;
		$length = 0;
		$hasBlock = false;

		while($j < count($parsed) && $depth > 0) {
			$jType = $parsed[$j]['type'];
			$jName = $parsed[$j]['name'];
			$jSmarty = $parsed[$j]['smarty'];
			$jPreserve = $parsed[$j]['preserve'];

			if($jPreserve) {
				$j++;
				continue;
			}

			if($jType === 'open' || $jType === 'smarty_open') {
				if(!$jSmarty && !in_array($jName, $inlineTags, true) && !in_array($jName, $textInlineTags, true)) {
					$hasBlock = true;
					break;
				}
				if($jSmarty && !in_array($jName, $smartyBlockTags, true)) {
					$hasBlock = true;
					break;
				}
				$depth++;
			}
			else if($jType === 'close' || $jType === 'smarty_close') {
				$depth--;
				if($depth === 0 && $jName !== $name) {
					$hasBlock = true;
					break;
				}
			}
			else if($jType === 'text' || $jType === 'smarty_inline' || $jType === 'void') {
				$length += strlen(trim($parsed[$j]['raw']));
				if($length > $maxInlineLength) {
					$hasBlock = true;
					break;
				}
			}
			$j++;
		}

		if(!$hasBlock && $depth === 0) {
			$ranges[] = [$i, $j, $name];
		}
	}

	$expanded = [];
	foreach($ranges as $r) {
		$start = $r[0];
		$end = $r[1];
		$name = $r[2];

		while($start > 0 && in_array($parsed[$start - 1]['type'], ['text', 'smarty_inline', 'void'], true)) {
			$prevRaw = $parsed[$start - 1]['raw'];
			if(strlen(trim($prevRaw)) > 20 || trim($prevRaw) === '') {
				break;
			}
			$start--;
		}

		while($end < count($parsed) && in_array($parsed[$end]['type'], ['text', 'smarty_inline', 'void'], true)) {
			$nextRaw = $parsed[$end]['raw'];
			if(strlen(trim($nextRaw)) > 20 || trim($nextRaw) === '') {
				break;
			}
			$end++;
		}

		$expanded[] = [$start, $end, $name];
	}
	$ranges = $expanded;

	sort($ranges);
	$merged = [];
	foreach($ranges as $r) {
		if(!empty($merged) && $r[0] <= $merged[count($merged) - 1][1]) {
			$prevName = $merged[count($merged) - 1][2];
			$currName = $r[2];
			if(in_array($prevName, $inlineTags, true) && in_array($currName, $inlineTags, true)) {
				$merged[count($merged) - 1][1] = max($merged[count($merged) - 1][1], $r[1]);
			}
			else {
				$merged[] = $r;
			}
		}
		else {
			$merged[] = $r;
		}
	}
	$ranges = $merged;

	$out = [];
	$level = 0;
	$i = 0;
	$buffer = '';

	$flushBuffer = function() use (&$out, &$buffer, &$level, $indent) {
		if($buffer !== '') {
			$buffer = preg_replace('/\s+/', ' ', $buffer);
			$buffer = trim($buffer);
			if($buffer !== '') {
				$out[] = str_repeat($indent, $level) . $buffer;
			}
			$buffer = '';
		}
	};

	while($i < count($parsed)) {
		$inRange = false;
		$rs = $re = null;
		foreach($ranges as $r) {
			if($r[0] <= $i && $i < $r[1]) {
				$inRange = true;
				$rs = $r[0];
				$re = $r[1];
				break;
			}
		}

		if($inRange) {
			$flushBuffer();
			$line = '';
			for($j = $rs; $j < $re; $j++) {
				$line .= $parsed[$j]['raw'];
			}
			$line = preg_replace('/\s+/', ' ', $line);
			$out[] = str_repeat($indent, $level) . trim($line);
			$i = $re;
			continue;
		}

		$type = $parsed[$i]['type'];
		$name = $parsed[$i]['name'];
		$raw = $parsed[$i]['raw'];
		$preserve = $parsed[$i]['preserve'];

		if($type === 'doctype' || $type === 'comment') {
			$flushBuffer();
			$out[] = str_repeat($indent, $level) . trim($raw);
			$i++;
		}
		else if($type === 'open') {
			$flushBuffer();

			if($i + 1 < count($parsed)
				&& $parsed[$i + 1]['type'] === 'close'
				&& $parsed[$i + 1]['name'] === $name) {
				$out[] = str_repeat($indent, $level) . trim($raw) . trim($parsed[$i + 1]['raw']);
				$i += 2;
			}
			else if($i + 2 < count($parsed)
				&& $parsed[$i + 1]['type'] === 'text'
				&& $parsed[$i + 2]['type'] === 'close'
				&& $parsed[$i + 2]['name'] === $name
				&& preg_match('/^\s+$/', $parsed[$i + 1]['raw'])) {
				$out[] = str_repeat($indent, $level) . trim($raw) . ' ' . trim($parsed[$i + 2]['raw']);
				$i += 3;
			}
			else if($preserve) {
				$out[] = str_repeat($indent, $level) . trim($raw);
				$level++;
				$i++;
				$content = [];
				while($i < count($parsed)) {
					if($parsed[$i]['type'] === 'close' && $parsed[$i]['name'] === $name) {
						$level--;
						break;
					}
					$content[] = $parsed[$i]['raw'];
					$i++;
				}
				if(!empty($content)) {
					$out[] = str_repeat($indent, $level) . implode('', $content);
				}
				if($i < count($parsed) && $parsed[$i]['type'] === 'close') {
					$out[] = str_repeat($indent, $level) . trim($parsed[$i]['raw']);
				}
				$i++;
			}
			else {
				$out[] = str_repeat($indent, $level) . trim($raw);
				$level++;
				$i++;
			}
		}
		else if($type === 'close') {
			$flushBuffer();
			$level = max(0, $level - 1);
			$out[] = str_repeat($indent, $level) . trim($raw);
			$i++;
		}
		else if($type === 'void') {
			$buffer .= trim($raw) . ' ';
			$i++;
		}
		else if($type === 'smarty_open') {
			$flushBuffer();

			if($i + 1 < count($parsed)
				&& $parsed[$i + 1]['type'] === 'smarty_close'
				&& $parsed[$i + 1]['name'] === $name) {
				$out[] = str_repeat($indent, $level) . trim($raw) . trim($parsed[$i + 1]['raw']);
				$i += 2;
			}
			else if($i + 2 < count($parsed)
				&& $parsed[$i + 1]['type'] === 'text'
				&& $parsed[$i + 2]['type'] === 'smarty_close'
				&& $parsed[$i + 2]['name'] === $name
				&& preg_match('/^\s+$/', $parsed[$i + 1]['raw'])) {
				$out[] = str_repeat($indent, $level) . trim($raw) . ' ' . trim($parsed[$i + 2]['raw']);
				$i += 3;
			}
			else {
				$out[] = str_repeat($indent, $level) . trim($raw);
				$i++;
			}
		}
		else if($type === 'smarty_close') {
			$flushBuffer();
			$out[] = str_repeat($indent, $level) . trim($raw);
			$i++;
		}
		else if($type === 'smarty_inline') {
			$buffer .= trim($raw) . ' ';
			$i++;
		}
		else if($type === 'smarty_preserve' || $type === 'smarty_comment') {
			$flushBuffer();
			$out[] = str_repeat($indent, $level) . trim($raw);
			$i++;
		}
		else if($type === 'text') {
			$buffer .= $raw;
			$i++;
		}
		else {
			$flushBuffer();
			$out[] = str_repeat($indent, $level) . trim($raw);
			$i++;
		}
	}
	$flushBuffer();
	return implode("\n", $out);
}
