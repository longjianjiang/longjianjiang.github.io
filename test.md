

public func dataTask(__: PMKNamespacer, with convertible: URLRequestConvertible) -> Promise<(data: Data, response: URLResponse)> {
	return Promise { dataTask(with: convertible.pmkRequest, completionHandler: adapter($0)).resume() }
}
